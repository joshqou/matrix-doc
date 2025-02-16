# MSC3659 - Invite Rules

This MSC proposes the creation of an optional account data state which allows users to control how invites directed at them
are processed by their homeserver.

## Proposal

### Glossery
- Inviter: The matrix user which has created the invite request.
- Invitee: The matrix user which is receiving the invite request.
- Invite request: An invite that the homeserver is to process. For Synapse, this would be handled by [`on_invite_request`](https://github.com/matrix-org/synapse/blob/develop/synapse/handlers/federation.py#L752).

### `m.invite_rules`

An invite rules state contains one required key.
- `"rules"`: An Array of `RuleItem`s. The Array must contain no more than the number of rules the homeserver is
willing to process.

#### `RuleItemAction`
A String-Enum that defines an action that the ruleset evaluator is to perform.

- `"allow"`: Allow the invite request, breaks from ruleset evaluation.
- `"deny"`: Reject the invite request.
- `"continue"`: Do not take any action and continue ruleset evaluation.

#### `RuleItem`
A RuleItem defines a Rule that can test against an invite request.

- `"type"`: Required String-Enum, must be one of the defined types below.
- `"pass":` A required `RuleItemAction` that will be performed if the rule evaluates as True
- `"fail":` A required `RuleItemAction` that will be performed if the rule evaluates as False

##### `m.user`
Validates as True if the Inviter MXID is equal to the defined `"user_id"`.
- `"user_id"`: Required String, a valid user id. This value may also be a glob

##### `m.shared_room`
Validates as True if the Inviter and Invitee are in the defined `"room_id"`.
- `"room_id"`: Required String, a valid room id. This value may also be a glob

##### `m.target_room_id`
Validates as True if the target room id is equal to the defined `room_id`.
- `"room_id"`: Required String, a valid room id. This value may also be a glob

##### `m.target_room_type`
Validation depends on the value of `room_type`.
- `"room_type"`: Required String-Enum.
  - `"room_type": "is-direct-room"`: Rule evaluates as True if the Invitee's membership state in the target room has `"is_direct"` set to True.
  - `"room_type": "is-space"`: Rule evaluates as True if the target room's `m.room.create` `type` is `"m.space"`
  - `"room_type": "is-room"`: Rule evaluates as True if the target room is not a direct room or a space.

##### `m.compare`
Compares information about the Invitee and Inviter. Behaviour depends on the value of `compare_type`
- `"compare_type"`: Required String-Enum.
  - `"compare_type": "has-shared-room"`: Evaluates as True if the Inviter shares at least one room with the Invitee.
  - `"compare_type": "has-direct-room"`: Evaluates as True if the Inviter has an active room defined in the Invitee's `m.direct` account data state. *Active is defined as "if both the Invitee and Inviter are present".*

#### Evaluation
Ruleset evaluation is performed before an invite request is acknowledged by the homeserver, invite rejection refers to rejecting the invite request in the form of returning a HTTP error to the Inviter's homeserver. Not to reject an invite request which has already been acknowledged (visible to the Invitee) by the homeserver.
Homeservers may choose to skip ruleset evaluation entirely if the Inviter is a homeserver admin.

- The Invitee's homeserver receives an invite request from the Inviter:
  - If the `"m.invite_rules"` account data state exists, then:
    - If `"rules"` is defined, then for each `RuleItem`:
      - Evaluate the `RuleItem` and save either the `"pass"` or `"fail"` `RuleItemAction` depending on the result.
      - If the `RuleItemAction` is:
        - `"allow"`, then: Break from the invite rules loop.
        - `"deny"`, then: Respond with `M_FORBIDDEN`.
        - `"continue"`, then: Continue for each.

*If the rules loop is iterated through without any action taken, it is treated as `"allow"`.*

Implementations may wish to utilise result caching where applicable to improve performance. Such as for rules that may require comparing the joined rooms of each user.

*Such cache would be storing the resulting `Boolean` returned during `RuleItem` evaluation, **not** the `RuleItemAction` which is picked from the defined `"pass"` or `"fail"` keys.*

#### Invite Rejection
If an invite is to be rejected, the homeserver *should* respond with M_FORBIDDEN, and the error message: "This user is not permitted to send invites to this server/user"

#### Capabilities
Invite Rules requires an additional capability entry at `client/r0/capabilities` to signal to clients how many rules
the homeserver is willing to process. The suggested maximum is 128. Homeservers are free to set their own maximum

```js
{
    "m.invite_rules": {
        "enabled": true,
        "maximum_rules": Integer
    }
}
```

#### Example
The following example will enforce the following:
- Any invites from `badguys.com` will be blocked.
- Invites from `@bob:example.com` will be allowed.
- Invites from `@alice:example.com` will be blocked.
- Invites from any user who is also in `!a:example.com` will be allowed.
- Invites from any user who shares a room with the Invitee will be allowed on the condition they are inviting them into a direct message room.

```js
{
    "type": "m.invite_rules",
    "content": {
        "rules": [
            {
                "type": "m.user",
                "user_id": "*:badguys.com",
                "pass": "deny",
                "fail": "continue"
            },
            {
                "type": "m.user",
                "user_id": "*.badguys.com",
                "pass": "deny",
                "fail": "continue"
            },
            {
                "type": "m.user",
                "user_id": "@bob:example.com",
                "pass": "allow",
                "fail": "continue"
            },
            {
                "type": "m.user",
                "user_id": "@alice:example.com",
                "pass": "deny",
                "fail": "continue"
            },
            {
                "type": "m.shared_room",
                "room_id": "!a:example.com",
                "pass": "allow",
                "fail": "continue"
            },
            {
                "type": "m.compare",
                "compare_type": "has-shared-room",
                "pass": "continue",
                "fail": "deny"
            },
            {
                "type": "m.target_room_type",
                "room_type": "is-direct-room",
                "pass": "allow",
                "fail": "deny"
            }
        ]
    }
}
```

## Alternatives
Currently, there is no way outside of homeserver-wide restrictions (mjolnir, anti-spam plugins), for users to control who can send them invites. While users can ignore single users to prevent them from sending them invites, this does little since a malicious user simply create another matrix account.

## Potential Issues
There is a potential denial of service for the `has-shared-room` and `has-direct-room` invite rules, as they require searching through all rooms a user is in, which could be a lot. This heavily depends on the homeserver's internals of course.

The `"rules"` Array's suggested maximum may not be suitable for resource-strained, or particularly large homeservers. Homeservers should make the maximum rules configurable for the homeserver admin.

## Unstable prefix
While this MSC is in development, implementations of this MSC should use the state type `org.matrix.msc3659.invite_rules`
