# Linking a person's account at a service (OAuth)

An app that reads a person's mail, calendar or files at a service (Google,
Microsoft and the like) needs a token from that service for that person.
Linking only lets the app reach the service's data. It never tells the app
who the person is: the ticket's `person` does that, and the service's
account never decides who someone is or what they may do
(`reference/platform.md`).

Every call the app makes to the service starts on the computer and goes
out, and the sandbox lets an app reach any server (`ncdev spec hostapi`,
"The app sandbox"). The hard part is getting the token, because nothing
can call into an app except the host.

## Which way, in order

### 1. The device grant: first, wherever the provider offers it

OAuth's device authorization grant (RFC 8628) needs no address for the
provider to send the person back to:

1. When the person asks to link, the app calls the provider's device
   authorization endpoint (an ordinary https call out). It gets back a
   `device_code`, a `user_code`, a `verification_uri` and a polling
   `interval`.
2. The app's page shows the user code and the verification URL as a link.
   The person opens the link, signs in at the provider and enters the
   code. They can also do this on their phone.
3. The app polls the provider's token endpoint at that interval. On
   `authorization_pending` it keeps going, on `slow_down` it waits longer,
   and on `access_denied` or `expired_token` it stops. Poll while the
   person's page is open and asking the app "linked yet?". If a code runs
   out, the person starts again.
4. The app keeps the tokens in the host's secrets store (below) and
   refreshes the access token itself. If the provider sends a new refresh
   token, the app replaces the stored one.

This works the same on every computer and in every NativeCluster window,
because the app makes only outgoing calls.

- Put the provider's URL itself in the link's `href` (`target="_blank"` is
  fine). The app's window opens a link the person clicks, or a
  `window.open` from their click, in the person's default browser. A page
  that sends itself to another site (`location.href = …`) goes nowhere.
- Never open a browser from the program (`webbrowser.open` and the like).
  The program runs in the sandbox, maybe on a computer that is not the
  person's.
- Check that the provider allows the device grant for the scopes the app
  needs, because some allow it for only some scopes. Also check whether it
  needs a client of a different type: that registration is the user's to
  make (stop and ask, below).
- Libraries: MSAL's `initiate_device_flow` and
  `acquire_token_by_device_flow`. For other providers, call the documented
  device endpoint over plain https.

### 2. A key the person pastes, where the service offers one

Some services give a person a token of their own: a bot's token, a
personal access token, an app password. The app's settings page takes the
token, checks it with one call to the service and keeps it in the secrets
store. This also needs no redirect and works on any computer. Use it only
where the service offers such a token for what the app does. Never ask for
a person's password at the service.

### 3. A redirect back to the app's own page: only on the computer it runs on

The host serves each app at an address of its own. When a provider sends
the person back to that address, the host passes the request to the app
with the person's ticket.

The redirect URL is

    https://127.0.0.1:<port><a path of the app's own>

`<port>` is 8490 plus the app's slot on that computer. The app gets the
lowest free slot when it first starts there and keeps it until it is
removed. So the URL is **not stable**: on another computer, or on the same
one after a remove and reinstall, the port may be anything from 8490 to
8499. Build the URL from the `Host` header of the request that starts the
link (the host passes the browser's own header on), and register every
URL it may be with the provider.

What it takes today:

- The provider's page opens in the person's default browser, because the
  app's window opens outside links there. So the redirect lands in that
  browser, not in the app's window.
- That browser must be signed in to NativeCluster on the same computer.
  The app's address passes nothing on from someone who is not signed in.
  It bounces the browser once through NativeCluster's own page to pick up
  a sign-in there. If NativeCluster has no sign-in in that browser, the
  person sees "Sign in to use …" and the app never gets the code.
- Only on the computer the app runs on. The app's address answers a
  browser on that computer, or a NativeCluster window that a joined
  computer signed in. A browser on any other computer gets a 403. So if
  the app runs on the main and the person uses it from a laptop, this way
  fails.
- The address is https under the computer's own certificate. Only
  NativeCluster's windows trust that certificate: nothing in NativeCluster
  makes a browser trust it.
- The provider must accept that exact URL (https at 127.0.0.1, on one of
  ten ports). That is the provider's rule, so check it.

The host's part is tested: a person sent back from another site with
`?code=` is reloaded from the app's own address, so their sign-in there
counts, and reaches the app with their ticket. The whole chain through a
person's own browser is not tested. Because of these limits, this way is
the user's decision (stop and ask, below), and you try it on a real Mac
before counting on it. As with any OAuth client, check `state` and use
PKCE.

## What doesn't work: a loopback listener

The app's old code listens on `127.0.0.1:0` for the provider's redirect.
So does any library's "local server" sign-in, such as google-auth-oauthlib's
`InstalledAppFlow.run_local_server` or MSAL's `acquire_token_interactive`.
None of these runs in NativeCluster:

- The sandbox lets an app listen only on unix sockets: its own
  `LB_APP_SOCKET`, and sockets in `NC_APP_DATA` or `NC_APP_SOCKETS`. It
  binds no TCP or UDP address, loopback included. Nothing calls into an app
  except the host.
- The app can't reach this Mac's own addresses either. To the sandbox,
  localhost means every address this Mac has.
- The program may not even run on the computer whose browser the person
  uses.

Replace the listener with way 1, or with 2 or 3 as above. Never with a port
of the app's own.

## Where to keep the tokens

- **In the host's secrets store**: `/host/v1/secrets/<name>` on the host
  socket, from host 0.4.15 (say `min_host` 0.4.15). The Python kit has
  `nativecluster.host().secret(name)`, `.set_secret(name, value)` and
  `.delete_secret(name)`. The Go kit's `HostClient` has `Secret`,
  `SetSecret` and `DeleteSecret`. From any other language, send `GET`,
  `PUT {"value": …}` or `DELETE` with `Authorization: Bearer
  <app_credential>` (`reference/contract.md`). The host keeps the values in
  NativeCluster's own database, sealed for this app and never in
  `NC_APP_DATA`.
- **A name** is 1 to 128 letters, digits and `. _ : -`. It is a label,
  never a secret itself.
- **Whose.** In an app made for each person (`"per_person": true`), each
  person's process has a store of its own, and the host deletes it along
  with the person. In any other app there is one store for the whole app.
  Put the ticket's person in each name (`google.token:p_3f…`), read only
  the asking person's values, and delete them yourself when the person
  unlinks: the host can't tell whose they are.
- **Caps**: 64 KiB for one value, 1 MiB for one store.
- **Unreadable** (`409 unreadable`; the kits' `SecretUnreadable` and
  `ErrSecretUnreadable`) means this computer can no longer open the value.
  Ask the person to link again, and keep the new token.
- **Never** keep tokens in a file in `NC_APP_DATA`, which every program of
  the person's can read and every backup holds. Never in the Keychain:
  the sandbox has no keychain services. Never in a log, and never the
  code either.
- "Remove app and its data" deletes them.

## On another computer

Each computer keeps its own. The app's secrets on one computer are not on
any other, and the id the app knows a person by is made on each computer.
The app runs on each computer where its person uses it, so a person links
once on each one. On a computer without a token yet, the feature says so
("Link your account on this computer") and offers the link. Everything
else in the app works meanwhile, and nothing waits for the computer where
they linked first.

The main's app has a store of its own too. Scheduled work that runs on the
main can use a person's account only through a token the main's app holds
(stop and ask, below).

## Stop and ask

Ask these with the rest of `reference/stop-and-ask.md`, all at once:

1. **No device grant and no key.** The provider offers only a redirect
   for what the app needs. The options are:
   - (a) the redirect to the app's own page, with each limit above said
     plainly;
   - (b) a web page the app's developer runs on the internet that receives
     the redirect and that the app then asks for the result (an outgoing
     call). It works on any computer, but the developer's server sees each
     person's code and has to be run and kept up;
   - (c) keep the feature off and say why.

   Recommend (a) for an app the person uses on their own computer, once a
   try on a real Mac works; otherwise (c).
2. **The provider's client registration**: a client of the device-grant
   type, or the redirect URLs for (a). The registration belongs to the
   user's or the developer's account at the provider. Ask for the client
   id; never make or reuse one yourself.
3. **Scheduled work on the main that needs the person's account.** The
   options are: the person links on the main too; the app hands the token
   to its main (`host().hand_to_main`); or the work runs without that
   account. Recommend linking where the work runs, so no token travels
   between computers.

Never invent a platform feature for this. NativeCluster does not link
accounts for apps, does not share one token across computers, and has no
callback address of its own for providers. If a plan needs one, it is a
question for the user, not something to build around.
