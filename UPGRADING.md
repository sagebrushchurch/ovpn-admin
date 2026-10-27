# Upgrading notes (Sagebrush fork)

History and lessons from moving this fork from the 2021 code to current `palark/ovpn-admin` `master` (October 2026), plus a checklist for next time.

## What changed in the October 2026 upgrade

* Rebased on `palark/ovpn-admin` `master` (the old `flant/ovpn-admin`). Upstream publishes only Helm chart tags, so track `master`, not releases.
* Base images: Alpine 3.24 (OpenVPN 2.7, easy-rsa 3.2, OpenSSL 3), Node 16 and Go 1.24 for the UI build. The old root `Dockerfile` is gone; the UI builds from `Dockerfile.ovpn-admin`.
* Security: upstream fixed a path traversal in static file serving (packr, #286/#442) and dropped packr. Years of npm and Go dependency updates are included.
* Local customizations (kept on top of upstream): see "Sagebrush fork notes" in `README.md`.

## Problems we hit, and the fixes

| Symptom | Cause | Fix |
| --- | --- | --- |
| New users never appear; the app logs `Certificate ... issued` anyway, with `exit status 1` | The PKI was created with easy-rsa 3.0. Its `pki/openssl-easyrsa.cnf` lacks `--copy-ext` support needed by easy-rsa 3.2. `configure.sh` only copies the packaged files when it creates a **new** PKI. The app ignores easyrsa's exit status and hides its output. | Refresh `pki/openssl-easyrsa.cnf` and `pki/x509-types/` from `/usr/share/easy-rsa/` (see checklist) |
| MikroTik (and other CBC-only) clients get `AUTH_FAILED, Data channel cipher negotiation failed (no shared cipher)` | OpenVPN 2.6+ only allows ciphers in `data-ciphers` (GCM and ChaCha by default); `cipher AES-128-CBC` no longer counts | `data-ciphers` and `data-ciphers-fallback` in `setup/openvpn.conf` (GCM preferred, CBC allowed) |
| New certificates valid for only 825 days | easy-rsa 3.2 default. `CLIENT_CERT_EXPIRATION_DAYS` only applies to the Kubernetes backend | `EASYRSA_CERT_EXPIRE: "3650"` on the `ovpn-admin` service |
| Test build overwrote production image tags, or the two stacks collided | Both stacks used project name `ovpn-admin`, tags `*:local`, and the same host ports | Use a separate folder or `-p <name>`, and re-tag the old images (`docker tag openvpn:local openvpn:old`) |
| `Permission denied` reading `easyrsa_master/pki` | Containers run as root and write the PKI as root | Use `sudo`, or work inside the container; do not `chown` it |

## Upgrade checklist

1. **Back up** `easyrsa_master/` and `ccd_master/` (`sudo cp -a`, to keep ownership and permissions). These are the only irreplaceable data; the images are rebuilt from the repo.
2. **Never test against production files.** Clone the branch to a new folder, copy the data, use a different project name (`-p`), different image tags, and different host ports (a `docker-compose.override.yaml` with `ports: !override`; needs Docker Compose 2.24+). Remember `/udp` on UDP ports.
3. **Tag the running images** for rollback: `docker tag openvpn:local openvpn:old` (also `ovpn-admin:local` and `ovpn-admin-nginx:latest`).
4. **Refresh easyrsa's config files** in the copied PKI (once per PKI, after the new stack is up):
   ```
   docker compose exec ovpn-admin sh -c '
     cd /mnt/easyrsa/pki &&
     cp openssl-easyrsa.cnf openssl-easyrsa.cnf.old &&
     cp /usr/share/easy-rsa/openssl-easyrsa.cnf . &&
     cp -r x509-types x509-types.old &&
     cp -r /usr/share/easy-rsa/x509-types/. x509-types/'
   ```
5. **Verify signing:** `docker compose exec ovpn-admin sh -c 'cd /mnt/easyrsa && easyrsa --batch build-client-full zz-check nopass | tail -3'` must end with `Certificate created`, and the certificate must be valid for 3650 days. Revoke `zz-check` afterwards.
6. **Test clients of every kind:** an existing client config, a newly created user, a revoked user (must be refused), and a **CBC-only router** (MikroTik). A Windows or GCM-capable client alone does not exercise the cipher setup.
7. **Check protocol and port:** the server runs UDP (`--proto udp` in `setup/configure.sh`, `7777:1194/udp` in compose, `OVPN_SERVER` ending in `:udp`). Upstream defaults are TCP, so re-check these after any merge from upstream.
8. **Cut over:** stop the old stack, copy the data, start the new one, and confirm site gateways reconnect (they carry the routes). Keep the old folder and `:old` images for a few days.
9. **Watch the logs** after cutover. Clients still holding sessions from the old server log `incoming packet authentication failed` for a minute or two until their keepalive expires. That is normal.

## Things to remember

* Upstream's `docker-compose.yaml` turns on password auth (`OVPN_AUTH`, `OVPN_PASSWD_AUTH`); this fork sets both to `false` to match how it is actually used.
* The app logs `Certificate ... issued` even when easyrsa fails (`main.go`, non-Kubernetes branch of user creation). When users "don't get created", run the `build-client-full` command by hand to see the real error.
* `client-to-client` and site-to-site routes pass through the cloud VM. OpenVPN encrypts on one thread, so throughput depends on single-core speed; shared-core `e2` VMs are slow. About 200 Mbit/s was measured on an `e2-medium`.
* The old `nginx/.htpasswd` basic-auth hash is still in git history; treat it as exposed and do not reuse that password.
* The UI has no login of its own, and upstream says it is for private networks only. Authentication is done by Nginx Proxy Manager in front of the `nginx` container, so make sure ports 80/443 on this stack are reachable only from the proxy and not from the internet.
