# Heroku buildpack for Varlock and Sorbet

Prepare a root Varlock schema before a subdir buildpack promotes a Rails app from a monorepo

This classic buildpack copies `.env.schema` to `$PROJECT_PATH/config/varlock/.env.schema`. It removes only the standalone `@plugin(varlock-sorbet)` and `@generateSorbetEnv(...)` directives. All variable definitions and validation rules remain unchanged. It uses the committed `lib/env_schema.rb` instead of generating Ruby code during boot

It does not copy `.env` or environment-specific value files, install packages, load secrets, or change Heroku config vars. Other plugin and import directives are preserved and need their own dependencies. The supported schema has the two Sorbet directives on separate lines, as shown below

```dotenv
# @plugin(varlock-sorbet)
# @generateSorbetEnv(path=api/lib/env_schema.rb, filter="#api", prefixesToRemove=["PUBLIC_"])
```

## Setup

Set `PROJECT_PATH` to the Rails subdirectory, such as `api`. Commit the generated `api/lib/env_schema.rb`

Add this buildpack **before** the existing subdir buildpack, leaving Ruby last. Pin the URL to a reviewed commit SHA

```sh
heroku buildpacks:add --index 1 https://github.com/akodkod/heroku-buildpack-varlock#COMMIT_SHA --app YOUR_APP
heroku config:set VARLOCK_SCHEMA_PATH=config/varlock --app YOUR_APP
```

Expected order:

1. `akodkod/heroku-buildpack-varlock`
2. `timanovsky/subdir-heroku-buildpack`
3. Any binary dependency buildpacks, such as ActiveStorage Preview and Chrome
4. `heroku/ruby`

The buildpack writes the schema inside the Rails subdirectory so it survives promotion to the slug root. It requires a nonempty relative `PROJECT_PATH` that resolves inside the build directory

Update Rails `config/boot.rb` before `Varlock.load_and_set!`:

```ruby
monorepo_root = File.expand_path("../..", __dir__)
schema_path = File.expand_path(ENV.fetch("VARLOCK_SCHEMA_PATH", monorepo_root), File.expand_path("..", __dir__))
env_vars = Varlock.load_and_set!(path: schema_path, filter: "#api")
```

Relative paths resolve against the Rails root. With no override, local development still loads the monorepo schema. Configure the path as a Heroku config var so it is available during Ruby build steps, release tasks, web boot, and worker boot

Changing config vars can restart an existing Heroku app. Prepare the code and configuration together as part of the release. This buildpack does not deploy the app or migrate legacy variable names. Required runtime variables must already be prepared

## Checks

```sh
bash test/run
```

Tests cover schema preservation, excluding value files, repeated execution, paths with spaces, subdir promotion, invalid paths, and missing generated code. They use temporary fixtures and do not contact Heroku

This buildpack follows the [classic Heroku Buildpack API](https://devcenter.heroku.com/articles/buildpack-api). It is intended for the [subdir buildpack](https://github.com/timanovsky/subdir-heroku-buildpack) and schemas using [Varlock](https://varlock.dev) with `varlock-sorbet`
