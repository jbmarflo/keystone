<!--[meta]
section: api
subSection: authentication-strategies
title: Password Auth Strategy
order: 1
[meta]-->

# Password Auth Strategy

Authenticates party (often a user) based on their presentation of a credential pair.
The credential pair consists of an identifier and a secret (often an email address and password).

### Usage

Assuming a list of users such as:

```js
keystone.createList('User', {
  fields: {
    username: { type: Text },
    password: { type: Password },
    // Other fields ..
  },
});
```

We can configure the Keystone auth strategy as:

```js
const authStrategy = keystone.createAuthStrategy({
  type: PasswordAuthStrategy,
  list: 'User',
  config: {
    identityField: 'username',
    secretField: 'password',
  },
});
```

Later, the admin UI authentication handler will do something like this:

```js
app.post('/admin/signin', async (req, res) => {
  const username = req.body.username;
  const password = req.body.password;

  const result = await this.authStrategy.validate({
    identity: username,
    secret: password,
  });

  if (result.success) {
    // Create session and redirect
    // ..
  }

  // Return the failure
  return res.json({ success: false, message: result.message });
});
```

### Config

| Option              | Type       | Default                                 | Description                                                               |
| ------------------- | ---------- | --------------------------------------- | ------------------------------------------------------------------------- |
| `identity`          | `String`   | `email`                                 | The field `path` for values that uniquely identifies items                |
| `secret`            | `String`   | `password`                              | The field `path` for secret values known only to the authenticating party |
| `protectIdentities` | `Boolean`  | `false`                                 | Protect identities at the expense of usability                            |
| `identityFilter`    | `Function` | See: (identityFilter)[#identity-filter] | Used to filter matched identities                                         |

#### `identity`

The field `path` for values that _uniquely_ identifies items.
For human actors this is usually a field that contains usernames or email addresses.
For automated access, the `id` may be appropriate.

#### `secret`

The field `path` for secret values known only to the authenticating party.
The type used by this field must expose a comparison function with the signature
`compare(candidateValue, storedValue)` where:

- `candidateValue` is the (plaintext) value supplied by the actor attempting to authenticate
- `storedValue` is a value stored by the field on an item (usually a hash)

The build in `Password` field type fulfils this requirements.

#### `protectIdentities`

Generally, Keystone strives to provide users with detailed error messages.
In the context of authentication this is often not desirable.
Information about existing accounts can inadvertently leaked to malicious actors.

When `protectIdentities` is `false`,
authentication attempts will return helpful messages with known keys:

- `[passwordAuth:identity:notFound]`
- `[passwordAuth:identity:multipleFound]`
- `[passwordAuth:secret:mismatch]`

As a user, this can be useful to know and indicating these different condition in
the UI increases usability.
However, it also exposes information about existing accounts.
A malicious actor can use this behaviour to _verify_ account identities making further attacks easier.
Since identity values are often email addresses or based on peoples names (eg. usernames),
verifying account identities can also expose personal data outright.

When `protectIdentities` is `true` these error messages and keys are suppressed.
Responses to failed authentication attempts contain only a generic message and key:

- `[passwordAuth:failure]`

This aligns with the Open Web Application Security Project (OWASP)
[authentication guidelines](https://www.owasp.org/index.php/Authentication_Cheat_Sheet#Authentication_Responses)
which state:

> An application should respond with a generic error message regardless of whether the user ID or password was incorrect.
> It should also give no indication to the status of an existing account.

Efforts are also taken to protect against timing attacks.
The time spend verifying an actors credentials should be constant-time regardless of the reason for failure.

#### `identityFilter`

A function used to restrict items matched for authentication. The function is passed the `identityField` and `identity` given at login. It should return a Keystone graphQL where statement. The default function for matching items is:

```javascript
const identityFilter = ({ identityField, identity }) => ({
  [identityField]: identity,
});
```

This returns items where the `identityField` matches the `identity` provided.

You may wish to modify this if you want to restrict users that can login to the AdminUI. This example restricts the authentication strategy to only match admin users:

```javascript
const authStrategy = keystone.createAuthStrategy({
  type: PasswordAuthStrategy,
  list: 'User',
  identityFilter: ({ identityField, identity }) => ({
    AND: [{ [identityField]: identity }, { isAdmin: true }],
  }),
});
```