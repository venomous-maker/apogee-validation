# apogee-validation

Validation for [Apogee](https://github.com/venomous-maker): a validator with
Laravel's rules and messages (and a few more), wildcard fields, conditional
rules, rule objects and closures, the database's `unique`/`exists`, files
and image dimensions, errors that render themselves, `req.validate()`, and
form requests that authorize and validate before a controller runs.

```typescript
import "std"
import "apogee-validation"

let data = req.validate({
    "email": ["required", "email", Rule.unique("users").ignore(user.id)],
    "password": ["required", "confirmed", Password.min(12).mixedCase().numbers()],
    "items": "required|array|min:1",
    "items.*.sku": "required|distinct",
    "items.*.qty": "required|integer|between:1,100",
    "coupon": "nullable|prohibited_unless:plan,pro",
    "avatar": "nullable|image|max:2048|dimensions:min_width=200,ratio=1/1",
})
```

## Form requests

```typescript
class StorePostRequest extends FormRequest {
    fn authorize() -> bool { return self.user != null }
    fn rules() -> HashMap<string, any> {
        return {"title": "required|string|max:200", "tags": "array|max:5", "tags.*": "string|distinct"}
    }
    fn messages() -> HashMap<string, string> { return {"title.required": "Every post needs a title."} }
    fn attributes() -> HashMap<string, string> { return {"tags.*": "tag"} }
    fn prepareForValidation() { self.merge({"slug": Str.lower(self.string("title"))}) }
    fn after(v: Validator) { if (taken(self.string("slug"))) { v.addError("title", "That :attribute is taken.") } }
}

class PostController extends Controller {
    fn store(req: StorePostRequest) -> Response {        // never called with an invalid request
        return Response.created(Post.create(req.validated()))
    }
}
```

A form request **is** a `Request`, sharing the parsed input of the request it
stands for. It needs no constructor; the container makes it for the action,
and the router validates it as it is made:

1. `prepareForValidation()`
2. `authorize()`, where false is a 403 (`failedAuthorization`)
3. the rules, with `messages()`, `attributes()`, `stopOnFirstFailure()`,
   `withValidator(v)` and `after(v)`
4. a `ValidationException` (`failedValidation`), sent back to `redirectTo()`
   with the `errorBag()`
5. `passedValidation()`

`validated()`, `safe()` (with `only`, `except`, `merge`) and
`validatedInput(key)` give what passed.

## The rules

Rules are a `"rule|rule:a,b"` string, or a list of strings, built rules,
rule objects and closures (a regex containing `|` can stay in a string:
everything after `regex:` is its pattern).

- **Presence:** `required`, `required_if`, `required_unless`,
  `required_if_accepted`, `required_if_declined`, `required_with`,
  `required_with_all`, `required_without`, `required_without_all`,
  `required_array_keys`, `present`, `present_if`, `present_unless`,
  `present_with`, `present_with_all`, `filled`, `missing`, `missing_if`,
  `missing_unless`, `missing_with`, `missing_with_all`
- **Prohibition:** `prohibited`, `prohibited_if`, `prohibited_unless`,
  `prohibited_if_accepted`, `prohibited_if_declined`, `prohibits`
- **Acceptance:** `accepted`, `accepted_if`, `declined`, `declined_if`
- **Flow:** `bail`, `nullable`, `sometimes`, `exclude`, `exclude_if`,
  `exclude_unless`, `exclude_with`, `exclude_without`
- **Types:** `string`, `integer`, `numeric`, `decimal:min[,max]`, `boolean`,
  `array[:keys]`, `list`, `json`, `date`, `date_format:Y-m-d`, `email`,
  `url[:schemes]`, `active_url`, `ip`, `ipv4`, `ipv6`, `mac_address`, `uuid`,
  `ulid`, `alpha`, `alpha_num`, `alpha_dash`, `ascii`, `lowercase`,
  `uppercase`, `hex_color`, `timezone`, `regex`, `not_regex`
- **Sizes** (numbers by value; strings by characters; arrays by count;
  files in kilobytes): `min`, `max`, `between`, `size`, `gt`, `gte`, `lt`,
  `lte` (a number or another field), `digits`, `digits_between`,
  `min_digits`, `max_digits`, `multiple_of`
- **Comparison:** `in`, `not_in`, `same`, `different`, `confirmed[:field]`,
  `starts_with`, `ends_with`, `doesnt_start_with`, `doesnt_end_with`,
  `contains`, `doesnt_contain`, `distinct[:ignore_case]`, `in_array:other.*`
- **Dates** (another field, `now`, `today`, `tomorrow`, `yesterday`, or a
  date): `after`, `after_or_equal`, `before`, `before_or_equal`,
  `date_equals`
- **Files:** `file`, `image`, `mimes`, `extensions`, `mimetypes`,
  `dimensions:width=,height=,min_width=,max_width=,min_height=,max_height=,ratio=3/2`.
  Sizes are read from the PNG, GIF, JPEG, WebP or BMP header, with no image
  library.
- **Database:** `unique:table[,column[,ignoreId[,idColumn]]]`,
  `exists:table[,column]`, asked through `Validator.presence`, a
  `PresenceVerifier` the framework connects to Orbit.

Built rules: `Rule.in(list)`, `Rule.notIn`, `Rule.unique(table,
column).ignore(id).where(col, v)`, `Rule.exists`, `Rule.requiredIf(fn)`,
`Rule.prohibitedIf(fn)`, `Rule.excludeIf(fn)`, `Rule.when(cond, rules,
otherwise)`, `Rule.anyOf([rules, rules])`, `Rule.dimensions("...")`,
`Rule.regex`, `Rule.dateFormat`, `Rule.check(fn(attribute, value, fail))`
(a closure), `Password.min(n).letters().mixedCase().numbers().symbols()`,
and `FileRule.image().maxKilobytes(n).extensions([...])`. Your own:
implement `ValidationRule` (plus `DataAwareRule` to see all the data, and
`implicit()` to run on missing fields), or `Validator.extend(name, fn,
message)` for a rule usable by name.

## Messages

Laravel's English messages, with `:attribute`, `:min`, `:max`, `:size`,
`:value`, `:values`, `:other`, `:date` and `:format` filled in. Custom ones
are keyed by `"field.rule"`, `"items.*.qty.rule"` or `"rule"`, and attribute
names by field or pattern. `MessageBag`: `has`, `first`, `get`
(wildcards too), `all`, `count`, `toMap()`.

## Errors

A `ValidationException` renders itself. A JSON client gets a 422 with
`{"message": "The email field is required. (and 2 more errors)", "errors":
{"email": [...], ...}}`. A form gets a redirect back (or to `redirectTo`)
with the errors flashed under their bag's name and the old input flashed,
minus `password`, `password_confirmation` and `current_password`. That
needs a session; without one it gets a 422 page listing the errors.

To fail without rules (a sign-in that does not match, say), throw
`ValidationException.withMessages({"email": ["These credentials do not match our records."]})`:
it renders the same way.

## Tests

```bash
grav test tests/validation.grav
```

The form-request test runs through `apogee-routing` (a sibling checkout).

MIT licensed.
