# RedCraft Manager Daemon config (rcmd_config)

This is the whole configuration of the RedCraft Minecraft network: every plugin we ship, every server template, and every pin saying which version goes where.

[rcmd](https://github.com/redcraft-org/RedCraftManagerDaemon) reads this repository and nothing else. If it's not in here, it's not on the servers.

## Requirements

Nothing, if you just want to read it. To check your changes locally you'll need rcmd itself, and `rcmd validate .` will tell you what's wrong.

## How it works

There are two things in here.

`catalog.yml` lists every jar rcmd knows how to fetch, and where from.

`templates/<name>/` is one server template: a `template.yml` saying what it inherits and what it pins, and a `config/` tree that gets copied onto the server.

### The catalog

Each entry says what the thing is and where it lives:

- `name` is what templates pin, and it becomes the file name on the server, so `CMI` lands as `plugins/CMI.jar`
- `kind` is `engine`, `plugin` or `mod`. An engine becomes `server.jar`, a plugin goes in `plugins/` and a mod goes in `mods/`
- `source` is `github`, `modrinth`, `jenkins`, `papermc`, `spiget`, `fabricmc`, `direct`, or `manual`
- `url` is the project page or API endpoint for that source
- `file_filter` picks one file out of a release that ships several, like `LuckPerms-Bukkit-*.jar`. `*` is the only wildcard, everything else is literal
- `archive_filter` does the same inside a zip, for entries that unzip first
- `post_processors` say how to read the version back out of the file. Almost always `plugin` or `fabric_mod`
- `enabled` can be set to false to keep an entry around without checking it

`source: manual` means there is nothing to download. A few plugins are paid and sit behind a bot check, so rcmd doesn't pretend it can fetch them, and somebody uploads the jar instead. They still get pinned, built and deployed like everything else.

### Templates

A template is a stack of layers. `hub` inherits `common_spigot`, which means everything `common_spigot` has, plus what `hub` adds on top.

Layers are applied parent first, and a child replaces a whole file rather than merging into it. If `hub` wants a different `server.properties` it ships the whole file. That's on purpose, because merging two YAML files and hoping the result is what either author meant is how you get a config nobody can explain.

- `inherits` names the parent layer
- `buildable: false` marks a layer that is not a server on its own, like `common_spigot`. Nobody can deploy it by accident
- `artifacts` maps a catalog name to a version pin
- `secrets` maps a placeholder to an environment variable

Pins are the same wildcards as the file filters. `5.5.*` takes any 5.5 release, `26.2` takes exactly that, and `*` takes whatever came in last.

### Secrets

There are no secrets in this repository, and there never will be.

A config file carries a placeholder like `MYSQL_PASSWORD` where the password goes, and the `secrets` block says which environment variable holds the real value on the machine doing the build. Substitution happens on the built output, never on the files in here.

:warning: If you add a config file with a password in it, that password is now public. Use a placeholder and add it to the `secrets` block instead.

## How to add a plugin

- Add an entry to `catalog.yml` with its `kind`, `source` and `url`
- Pin it in whichever `templates/<name>/template.yml` should get it
- Run `rcmd validate .` to check you didn't typo the name
- Open a pull request

A pin that names something the catalog doesn't have will fail validation. That's deliberate: the tooling this replaced would happily build a template pinning a plugin nobody had ever downloaded, and you'd find out at restart time.

## How to change a version

You usually don't have to. rcmd checks every night, and anything matching the existing pin is picked up on its own.

A release that moves outside the pin (say `5.6.0` when the pin says `5.5.*`) gets its own pull request, because that's a decision somebody should make rather than a surprise at 3am. Merging it changes nothing by itself, the next build picks it up and the next restart window ships it.

## Contributing

You are free to suggest changes by opening an issue ticket.

You can also open PRs, remember to run `rcmd validate .` before opening a pull request (the CI does it too, but finding out locally is faster :))
