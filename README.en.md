# kaist-openconnect

[한국어](README.md) | **English**

A small, dependency-free command-line client for connecting to the **KAIST VPN**
(Ivanti Connect Secure + AirCUVE "2Auth" two-factor) entirely from the terminal —
no browser required.

It prompts for your ID, password, and one-time code (SMS or email), completes the
two-factor handshake, and then brings the tunnel up with
[OpenConnect](https://www.infradead.org/openconnect/).

## Install

Paste this one line into your terminal — it installs everything you need
(Homebrew on macOS if missing, plus OpenConnect) and links `kvpn` onto your
`PATH`:

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/predict-woo/kaist-openconnect/main/install.sh)"
```

Then just run `kvpn`. It skips anything already installed, so it's safe to
re-run — re-running also updates kvpn to the latest version. On Linux it uses
your distribution's package manager (apt/dnf/pacman/zypper) for the
dependencies.

<details>
<summary>Manual install (if you'd rather not run the installer)</summary>

Install the [requirements](#requirements) yourself, then:

```sh
git clone https://github.com/predict-woo/kaist-openconnect.git
cd kaist-openconnect
chmod +x kvpn
./kvpn
```

Optionally put it on your `PATH`:

```sh
ln -s "$PWD/kvpn" /usr/local/bin/kvpn
```

</details>

Set a default ID so you can just press Enter at the prompt:

```sh
export KVPN_USER=your_id   # add to ~/.zshrc or ~/.bashrc
```

![A full kvpn session: ID/password/realm prompts, SMS one-time code, Keychain storage offer, and OpenConnect bringing up the tunnel](docs/screenshot.png)

On macOS, running `kvpn` retrieves the email code from Mail automatically and
connects with hotspot-compatible settings. Both the KAIST password and the local
`sudo` password are requested before sending the code. A macOS notification with
sound appears when OpenConnect reports that the tunnel is connected.

## Why

KAIST's VPN uses Ivanti Connect Secure with an AirCUVE second factor whose OTP
step runs inside a JavaScript portal on a separate host. OpenConnect on its own
can't drive that portal, so the usual advice is "log in with a browser, copy the
`DSID` cookie, then run openconnect." This tool automates that whole dance: it
performs the portal's OTP flow over plain HTTP and hands the resulting session
cookie to OpenConnect for you.

**It does not bypass anything.** You still authenticate with your own password and
a fresh one-time code on every connection — exactly like the official client.

## Requirements

- **Python 3** (standard library only — no `pip install`)
- **[OpenConnect](https://www.infradead.org/openconnect/)** on your `PATH`
  (`brew install openconnect` on macOS)
- `sudo` rights (OpenConnect needs root to create the tunnel interface)

Developed and tested on macOS. It should work on Linux with OpenConnect's default
`vpnc-script`; reports welcome.

## Usage

Run `./kvpn` and follow the prompts. To disconnect, press **Ctrl-C** in the
terminal running it — OpenConnect tears down the tunnel and restores your routes.

### Hotspots, automatic email codes, and notifications

```sh
./kvpn                      # macOS: automatic Mail code + hotspot settings
./kvpn --sms                # enter an SMS code manually
./kvpn --email --manual-code # enter an email code manually
./kvpn --standard-network   # allow UDP and the server's default MTU
./kvpn --help
```

- **Hotspots:** the default uses `--no-dtls --mtu=1280 --base-mtu=1400` to
  accommodate mobile networks that block UDP/ESP or have smaller MTUs. These
  settings also work on Wi-Fi. Use `--standard-network` to restore UDP and the
  server's MTU. This does not resolve every possible hotspot issue, such as DNS
  failures or blocked login portals.
- **Automatic email codes:** add the receiving account to macOS Mail. Allow your
  terminal to control Mail when prompted; manage this under **System Settings →
  Privacy & Security → Automation**. Only new messages in the inbox received
  after this request qualify. The default requires a sender at `@kaist.ac.kr`,
  VPN context, and a labeled code. Existing messages and ambiguous codes are
  ignored. Codes are neither printed nor saved. For another sender, set its
  exact address with `export KVPN_MAIL_SENDER=actual-address@example.com`.
  Keep these messages in the inbox if you use mail filtering rules. Mail access
  failures or a 120-second timeout fall back to manual entry.
- **Password order:** KAIST credentials and local `sudo` authorization are
  collected before requesting the code. Authorization is refreshed while
  waiting; launching the tunnel will not prompt for another password.
- **Notifications:** a macOS notification with sound appears after OpenConnect's
  `Connected as …` message. Check notification settings and Focus if it does
  not appear. Notification failures do not interrupt the VPN.

Linux retains the SMS/Email selection and manual code entry. The hotspot settings
and early password prompts also apply on Linux; Mail automation and macOS
notifications are macOS-only.

### Stored credentials

After a successful login, kvpn offers to store your credentials so future runs
skip the KAIST password prompt. The password goes into the
**macOS Keychain** (service `kvpn`); your ID and realm go into
`~/.config/kvpn/config.json` (user-only, `0600`). On systems without a
Keychain, the password is kept in that same `0600` file instead (kvpn warns
you first). A fresh one-time code is still required on every connection.

To remove everything that was stored:

```sh
./kvpn --forget
```

Set `KVPN_DEBUG=1` to print the internal HTTP redirect chain (useful if the login
flow changes and something breaks):

```sh
KVPN_DEBUG=1 ./kvpn
```

## How it works

The login is a chain across two hosts — the Ivanti gateway (`kvpn.kaist.ac.kr`)
and the AirCUVE portal (`kvpnportal.kaist.ac.kr:8443`):

1. `GET kvpn.kaist.ac.kr/` — obtain the Ivanti sign-in session.
2. `GET portal /view/redirectVPN` — mint a portal session and correlation code.
3. `GET portal /view/onepassCheck` → `/vpnInit` — set the OTP context.
4. `GET portal /oauth/authorize` — register the OAuth request as Spring Security's
   "saved request" so the post-OTP redirect returns to the OAuth flow.
5. `POST portal /api/portal` (`cmd=AuthPortal.twoAuthSend`) — send the SMS/email code.
6. `POST portal /j_spring_security_check.do` — verify the code; Spring redirects
   through `/oauth/authorize` → `/vpn/<client_id>?code=…`.
7. That page exposes a `message` token, which becomes Ivanti's `password#2`.
8. `POST login.cgi` with username, password, realm, and `password#2` → a `DSID`
   session cookie.
9. `sudo openconnect --protocol=nc --cookie-on-stdin kvpn.kaist.ac.kr` — the DSID
   is passed on stdin so it never appears in the process list.

## Notes & troubleshooting

- **Credential storage is opt-in.** Nothing is saved unless you answer "y" at
  the store prompt; `kvpn --forget` removes it all. The `DSID` is ephemeral
  either way (it dies when you disconnect), and a fresh one-time code is
  required on every connection.
- **Stored password went stale?** If your KAIST password changes, the final
  login step will fail — run `kvpn --forget` and log in again.
- **`sudo` authorization failure:** authorization is checked before sending a
  code. Verify your local login password and administrator rights, then retry.
- **Harmless route warnings:** on connect you may see two lines like
  `Can't assign requested address` / `File exists` for a route to your own VPN IP
  or an already-present route. Every other route still installs; this doesn't
  affect connectivity.
- **If login suddenly breaks**, KAIST likely changed the portal flow. Run with
  `KVPN_DEBUG=1` and open an issue with the (redacted) hop list.

## Disclaimer

This is an unofficial, personal interoperability tool. It is **not affiliated with
or endorsed by KAIST, Ivanti, or AirCUVE**. Use it with your own account and in
accordance with your institution's acceptable-use policy. Provided "as is" without
warranty of any kind (see [LICENSE](LICENSE)).

## Acknowledgements

Built on top of the excellent [OpenConnect](https://www.infradead.org/openconnect/)
project, which does the actual VPN tunneling.
