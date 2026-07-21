---
description: "Change the FastSvelte password policy: adjust the length limits, add your own rules with a Pydantic validator, and mirror them in the frontend forms."
keywords: "fastsvelte password policy, password validation, pydantic validator, custom password rules"
---

# Changing the Password Policy

## What's enforced now

Passwords must be **8 to 64 characters**, checked on signup, invitation accept and
password reset. Both the API and the forms enforce it, so a request that skips the
browser is rejected too.

Login and the current-password field on the profile page are the exception: they have a
maximum but no minimum. They check an existing password rather than set a new one, so one
that's too short should come back as a failed login, not as a form error announcing how
long passwords have to be.

## Changing the length

Both limits are constants in `backend/app/model/common.py`:

```python
PASSWORD_MIN_LENGTH = 8
PASSWORD_MAX_LENGTH = 64
```

Raising the maximum is safe. Lowering it is the change worth thinking about: password
managers generate 20 characters or more by default, and a passphrase like
`sturdy walnut harbor lamp` is 25, so a low cap turns away the strongest passwords your
users have.

The frontend forms carry the same numbers and need updating too.

## Adding a rule of your own

Attach a validator to `NewPassword` in `backend/app/model/common.py`. This one requires a
mix of character types, the rule most often asked for:

```python
from pydantic import AfterValidator, Field

def _require_character_classes(value: str) -> str:
    missing = []
    if not any(c.islower() for c in value):
        missing.append("a lowercase letter")
    if not any(c.isupper() for c in value):
        missing.append("an uppercase letter")
    if not any(c.isdigit() for c in value):
        missing.append("a digit")
    if all(c.isalnum() for c in value):
        missing.append("a symbol")
    if missing:
        raise ValueError("must contain " + ", ".join(missing))
    return value

NewPassword = Annotated[
    str,
    Field(min_length=PASSWORD_MIN_LENGTH, max_length=PASSWORD_MAX_LENGTH),
    AfterValidator(_require_character_classes),
]
```

Collecting the missing pieces before raising means the user is told everything at once
("must contain an uppercase letter, a digit, a symbol") rather than fixing one and
discovering the next.

!!! info "Why not `Field(pattern=...)`?"

    The usual regex trick, `(?=.*[A-Z])`, uses lookahead, and Pydantic's default regex
    engine doesn't support it. The model then fails to build at import time rather than
    at validation time, so the whole app stops starting. Use a validator instead.

!!! warning "Add rules to `NewPassword` only, never to `SubmittedPassword`"

    `SubmittedPassword` is for passwords being *checked*, at login and in the
    current-password field. A rule there rejects anyone whose existing password predates
    it, and turns a normal failed login into a validation error.

## Mirroring it in the frontend

The forms validate before submitting, so a rule added only on the backend leaves the user
stuck. Worse, they can't tell why: validation errors arrive as
`{"message": "Invalid request data", "details": {...}}`, and the forms display `message`.
So the browser says "Invalid request data" while the actual reason sits in `details`.

Update the zod schema in each form that sets a password:

| Form | File |
|---|---|
| Signup | `frontend/src/routes/(auth)/signup/+page.svelte` |
| Invitation accept | `frontend/src/routes/(auth)/invite/accept/+page.svelte` |
| Reset password | `frontend/src/routes/(auth)/reset-password/+page.svelte` |
| Change password | `frontend/src/routes/(protected)/profile/+page.svelte` |

```ts
password: z
	.string()
	.min(8, 'Password must be at least 8 characters')
	.max(64, 'Password must be at most 64 characters')
	.regex(/[a-z]/, 'Password must contain a lowercase letter')
	.regex(/[A-Z]/, 'Password must contain an uppercase letter')
	.regex(/[0-9]/, 'Password must contain a digit')
	.regex(/[^A-Za-z0-9]/, 'Password must contain a symbol')
```

Zod records every rule that failed, and the form shows them one at a time as the user
types.

Leave `frontend/src/routes/(auth)/login/+page.svelte` alone, for the same reason as
`SubmittedPassword`.

## Why there are no character rules by default

Requiring an uppercase letter and a symbol sounds stricter, but it mostly produces
`Password1!`. Length is what actually costs an attacker time, and a rule that blocks
`sturdy walnut harbor lamp` while allowing `Password1!` is working against you. Current
guidance from NIST, the US standards body whose recommendations most of the industry
follows, is to require length, allow everything else, and drop composition rules entirely.
That's the default here. Turning them on is a decision about your own users and any
compliance regime you answer to, which is why it's a few lines rather than a setting.
