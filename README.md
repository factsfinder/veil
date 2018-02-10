# Veil

Simple passwordless authentication for your Phoenix apps.

## Installation

1. Create a new Phoenix project, and change to the working directory:

```shell
mix phx.new your_app
cd your_app
```

2. Add `veil` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [{:veil, "~> 0.1.0"}]
end
```

3. Fetch dependencies, and install Veil using the default task:

```shell
mix deps.get
mix veil.add
```

4. Update the Veil configuration in your `config.exs` file to add an API key for your preferred email service (for more details on the email options, please refer to the [Swoosh Documentation](https://github.com/swoosh/swoosh)).

```elixir
config :veil, YourAppWeb.Veil.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: "SG.your-api-key"
```

5. Launch your server and open http://localhost:4000/ in your browser.

```shell
mix phx.server
```

If you click the sign-in link in the top right and enter your email address, you'll be sent an email with a sign-in button. Click this to re-open the website and you'll see you are now authenticated.

## Configuration

The full configuration section added to your `config/config.exs` looks like this:

```elixir
config :veil,
  site_name: "Your Website Name",
  email_from_name: "Your Name",
  email_from_address: "yourname@example.com",
  sign_in_link_expiry: 3_600,
  session_expiry: 86_400 * 30,
  refresh_expiry_interval: 86_400

config :veil, YourApp.Veil.Scheduler,
  jobs: [
    # Runs every midnight to delete all expired requests and sessions
    {"@daily", {YourApp.Veil.Clean, :expired, []}}
  ]

config :veil, YourAppWeb.Veil.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: "SG.your-api-key"
```

You should move the third part of this to a file that is not under version control, or save your API key as an environment variable instead.

## Why Passwordless?

Most users choose insecure passwords, and for most use cases it's safer and easier to have them click a link in an email to authenticate themselves. An added advantage is that the lack of a password prevents the website leaking user passwords in the event of a data breach/hack.

The username/password paradigm is going to gradually die and the sooner the web can move past this for simple websites the better - Veil is my attempt to help speed this process up.

## Authenticating Routes & Customising Veil

Virtually all of Veil's code is copied directly into your project when you run `mix veil.add`, so you can customise it to your heart's content.

By default `Plugs.Veil.UserId` is added to your default Router paths, which assigns the client's user_id to `conn.assigns[:veil_user_id]`. If you extend the `Veil.User` schema, you may want to add `Plugs.Veil.User` as well, which will assign the full `Veil.User` struct to `conn.assigns[:veil_user]`.

Authentication can either be handled using scopes in the Router, or in your Controllers. On adding Veil to your project, it adds the following block to your Router. Paths in this block will only be accessible to logged in users.

```elixir
# Add your routes that require authentication in this block.
# Alternatively, you can use the default block and authenticate in the controllers.
# See the Veil README for more.
scope "/", YourAppWeb do
  pipe_through([:browser, YourAppWeb.Plugs.Veil.Authenticate])
end
```

To restrict access in controllers, you can add `Plugs.Veil.Authenticate` like so:

```elixir
defmodule YourAppWeb.Controller
  use YourAppWeb, :controller

  plug YourAppWeb.Plugs.Veil.Authenticate when action in [:new, :create]

  # ...
```

This would restrict access to the new and create paths to logged in users only.

## Security

The sign-in/auth process works as follows:

1. User requests to sign-in using their email address. If they didn't previously have an account, one is created.

2. An email is then sent to the user with a sign-in link. The email and website response is identical whether or not they previously had an account - this is to avoid leaking information to attackers.

3. The link contains a Base32 encoded request id that is stored in a database by the server. When the user clicks the link, the server checks in the database to find out which user made the request, and how long it has been since they first tried to sign-in (default sign\_in\_link_expiry is 1 hour).

4. If the link hasn't expired, a new session is created for the user along with a new Base32 encoded session id, and this is saved to a cookie. The browser sends the session id to the server for each web request, and the server checks it hasn't expired. If it is older than the refresh\_expiry\_interval (default 1 day), then the session is extended until the session_expiry (default 30 days).

5. Request/Session ids are generated by concatenating together the following, calculating the SHA256 hash and then encoding to Base32. This should give sufficient entropy and length to be unguessable by an attacker.
  - A random string of bytes
  - The user's IP address
  - The user's user-agent string
  - A timestamp
  - A server secret that resets on startup

If you have any suggestions or questions about this process, please let me know.

## Phoenix APIs

Veil works for Phoenix APIs as well, although in order for it to work seamlessly you would need to use it as the backend to a mobile app where you can deep link the sign-in link sent via email to cause the app to open on the client's mobile device. That app can then post the request id to the server and receive a session id in return. The session id can then be sent along with future API requests by the mobile app.

Note that by default Sendgrid changes links to redirect via their domain, so you would need to change mail adaptor or set up whitelabelling in order for deep links to work.

If your phoenix app was created with the `--no-html` flag, then Veil will install it's own `no-html` version.

#### Example

```shell
$ curl --data "email=you@domain.com" http://localhost:4000/veil/users/
{"ok":true}
```

Email is sent containing the following URL:
http://localhost:4000/veil/sessions/new/DK6VNMHPNRVDVEFLVTYAGXKSHBXEIXK5WTY7O2RG6RC3STDQLAZA====

Your mobile app intercepts this URL after the user taps it, strips out the request id and then posts it back to the server to receive a session id in return:

```shell
$ curl --data "request_id=DK6VNMHPNRVDVEFLVTYAGXKSHBXEIXK5WTY7O2RG6RC3STDQLAZA====" \
http://localhost:4000/veil/sessions/new/
{"session_id":"BICJZMZHQY3UOL7SVUJNHGBJYUI2ZSE47MV2T6NZZNQUJ3UHIPCQ===="}
```

The session id will then need to be sent for future requests to protected api routes.

Any thoughts on how this could be easily extended to work better for APIs out-of-the-box are welcome.

## But I need something more full-featured!

Use [Coherence](https://github.com/smpallen99/coherence) or [Guardian](https://github.com/ueberauth/guardian).

## License

`Veil` is Copyright (c) 2018 Zander Khan

The source is released under the MIT License.

Check [LICENSE](LICENSE) for more information.
