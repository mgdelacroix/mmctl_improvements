# MMCTL improvements proposal

## Goals

The goal of this proposal is to outline and describe a set of changes
to apply on the next iteration of mmctl, and the principles that
motivate and guide those changes.

What started as specific changes closely related to the mmctl's
implementation has grown to contain suggestions and good practises
applicable to any CLI tool, so a possible byproduct of this analysis
could be a style guide that we could apply not only to mmctl itself
but to any other Mattermost CLI tool.

## Command structure

The CLI commands will follow the "topic-action" structure. When
thinking on how to reach a subcommand, we should think first on what's
the main topic that the subcommand is affecting, and then the action
that the command performs. Topics will generally be entities that the
command refers to, and actions would be verbs.

For example, in the case of the "create a user" command, the main
entity is `user`, and the action is `create`, so the final command
will be `mmctl user create`.

 - If topics or actions are composed of multiple words, we will
   separate them with dashes (for example `delete-all`).
 - Topics can be stacked if they make sense (like in `mmctl team
   member invite`), but this kind of nesting should be very obvious
   and not abused, as it harms command discoverability.

The alternative to this design is the "action-topic" structure, for
the "create a user" example it would be `mmctl create user`. This
model is very natural when there are only a few entities and actions,
but it gets very complicated when the CLI grows, which is the case for
mmctl specifically.

## Arguments and flags

To pass information to a command we can use both arguments and
flags. Arguments have the advantage of being direct; you don't have to
remember the "name" of the argument, just its purpose. For the case of
flags you do need to remember the name, but that could be an advantage
when you have plenty of different pieces of information to pass. On
the other hand, flags are easier to document, you can provide default
values and type checks with ease and flag names is not a problem if
you have autocompletion enabled.

So these are a few hints for choosing between both:

 - A mandatory requirement for args is that they have to be
   obvious. If the command moves something and requires of two args,
   it is obvious that those will be the source and the destination. If
   the command is creating a user and requires of four, what the first
   flag would be? the user name maybe? and the rest?
 - As a follow up for that first hint, args are fine if you plan to
   use one, maybe two. For three or more, it is better simply not to
   use them. Flags will provide with a much more flexible and clear
   method for those cases. Going back to the example of the user
   creation, each property that it requires can be a flag, and the
   help will specify what's the purpose of each flag, if they are
   required or not and if they have any type or default value.

When thinking about flags, take into account as well these principles:

 - Always use the short/long options. Long names should make as
   obvious as possible what each flag is, and short options will help
   experienced users go faster if they already know a command.
 - Provide sane default values for flags whenever possible.
 - Try to keep flags consistent; every command that lists an entity
   should have the same `--size` `--page` and `--all` flags, rather
   than accepting `--page` for one and `--offset` for the next.

Specific to the `cobra` library, there are a few helpers to help with
flags. We can use the `P` variant when declaring flags to use the
short/long form (`Flags().StringP` instead of `Flags().String`) and to
get automatic type checking; we can enforce the usage of some flags
with `MarkFlagRequired` and register dynamic autocompletion for a
flag's value with `RegisterFlagCompletionFunc`. If a flag is used in
several commands, we can create a function that attaches it to a
command and reuse it with ease.

## Progressive discovery

We should facilitate as much as possible the discovery of the tool
possibilities and features in an easy and natural way.

### Automatic suggestions

If the user tries to run a command that doesn't exists for our tool,
we should try to suggest commands that were close to what they typed,
in case there is a typo in the input:

```sh
$ mmctl uesr
Error: unknown command "uesr" for "mmctl"

Did you mean this?
        user

Run 'mmctl --help' for usage.
```

and in case there is a command that performs the same action that the
user was looking for, but with another name:

```sh
$ mmctl user remove
Error: unknown command "remove" for "user"

Did you mean this?
        delete

Run 'mmctl user --help' for usage.
```

### Manual follow up suggestions

There are some times when a use case involves several commands
executed one after another. For those kind of cases, we can make some
suggestions to the user when a command is run with the human readable
output, so they know what commands could be run after the one they
just finished executing.

A good example of this case is the bot creation flow, where most of
the times after creating a bot the user wants to create as well an
authentication token for it. The output could look like the following:

```sh
$ mmctl bot create --username calendarbot ...
Bot "calendarbot" created successfully

Use "mmctl bot generate-token" to create an authentication token for the bot
```

### Intelligent autocompletion

Currently we offer completions for `bash` and `zsh`, but those are
limited to the commands/subcommands of the tool and to their
flags. For the next mmctl iteration we should create a collection of
flags and arguments that the tool can provide autocompletion for and
structure them in a reusable way, so we can easily use them in the
commands that need them.

```go
func TeamUsersAddCmd() *cobra.Command {
    cmd := &cobra.Command{...}

    # This fn will attach the --team flag to the cmd and its autocompletions
    flags.AddTeamFlag(cmd)
    # then we mark it as mandatory if we need it to be
    cmd.MarkFlagRequired("team")

    return cmd
}
```

Luckily for us, the autocompletion functions receive a
`*cobra.Command` parameter that contains the parsed args that the user
has already written, which means that we can detect if the command
that we are autocompleting contains the `--local` flag and
autocomplete using the local path.

Sadly, this doesn't work correctly with `viper` at the moment, so if
we want to take into account the environment variable that enables
local mode, we have to independently fetch both values:

```go
cmd.RegisterFlagCompletionFunc("team", func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
	// viper doesn't take into account if the --local flag is in
	// use, so we have to retrieve both and combine them
	localF, _ := cmd.Flags().GetBool("local")
	localE := viper.GetBool("local")

    local := localF || localE

    ...
})
```

### Directed help if a requirement is not met

For the cases where an obvious requirement is not met when running a
command, we should not only indicate what the user needs to do, but
provide an example on how it is done:

```sh
$ mmctl user list
Error: cannot read user credentials: stat /home/mgdelacroix/.mmctl: no such file or directory

You need to log into an instance to use this command. To do so, you can use:

    mmctl login http://myinstance.com --username john --password secret --name myinstance

Please run "mmctl login --help" for usage.
```

A way to do this would be to use a centralised function to manage an
error and exit, and combine that with specific error types.

If your command fails with an `AuthError`, when you send that to the
`ErrorAndExit` helper, it will detect the type of error, print the
appropriate suggestion for the user to login and then exit.

### Command examples

On this same direction, we should provide the user with multiple rich
examples of usage for our commands and their different options:

```sh
$ mmctl login --help
Login into an instance and store credentials

Usage:
  mmctl login [instance url] --name [server name] --username [username] --password [password] [flags]

Examples:
  # if you only use the instance url, mmctl will prompt for the rest of the parameters
  $ mmctl auth login https://mattermost.example.com
  Connection name: local-server
  Username: sysadmin
  Password: mysupersecret

  # a username & password login will require of name, username and password
  $ mmctl auth login https://mattermost.example.com --name local-server --username sysadmin --password mysupersecret

  # if you use a MFA token, you will have to provide it as well
  $ mmctl auth login https://mattermost.example.com --name local-server --username sysadmin --password mysupersecret --mfa-token 123456

  # you can login with a personal access token too
  $ mmctl auth login https://mattermost.example.com --name local-server --access-token myaccesstoken
```

### Help

These are some things that we can do to improve how help is currently
done in mmctl:
 - We can assume that help is always shown as a result of a user
   request rather than as part of a script or automation, so we can
   safely apply things like colors to it.
 - We should consistently show which commands can be run in local mode
   and which don't in the command's help. If we decide to create our
   own struct type on top of `cobra.Command`, this could be solved
   using a `Local` boolean property.
 - We need a way to document local mode as a whole inside `mmctl`'s
   help. There is no specific way within `cobra` to do this, but we
   can create an "empty command" (as in a command with no `Run`
   function) that would only contain a general help explaining what
   local mode is and how it works. This way a user can run `mmctl
   local` or `mmctl help local` to find out what local mode does.

## Output format

Each command run can communicate through several ways with the user,
other commands and the shell itself; we have the exit code, the out
and err descriptors and the format of the output itself. Depending on
how the command is called, we should adapt those mechanisms to
provide with a consistent and useful output.

 - We should allow for at least three different output formats: human
   readable, JSON and table/tabular, which are useful depending on the
   context that the command is run. This can be done in `mmctl`
   modifying the `printer` package to add the table format and modify
   the others if needed.
 - We are overusing the `RunE` and `return error` pattern in
   `cobra`. Every time we return an error in a command, its usage gets
   printed, which is only useful for the cases of a bad input, so we
   should only return error in those cases, and create a helper like
   `ErrAndExit` that would print error information and exit with the
   proper status code. This would allow us as well to have a common
   function to process all errors and append useful information or
   follow up commands to them in a centralised way.
 - The default output format should always be the human readable one,
   that doesn't need to be easy to parse, can be colored and can
   contain things like recommendations and follow up commands.
 - We should expose a flag and an environment variable to disable
   colors in the output, and we should gracefully degrade to the
   colorless state if we detect that the user is running the command
   in a terminal that doesn't support it or as part of a pipe.

## Command factory

This is a section tied to `cobra` and how it handles commands. Right
now we're creating the command structs as package variables, and
that's limiting both how we are organising the code and our ability to
test it.

The way that we've structured the code, we're managing all the
command's flags of the same file in the `init` function of the file,
which is not ideal code organisation wise. This same decision and the
fact that cobra doesn't have a mechanism to "clean" a command after
its usage makes it difficult for us to create tests. Right now, we're
creating empty commands and attaching flags with the values we want
set as the default values to be able to correctly test them.

The proposed change here is to go for a "factory" approach:

```go
func LoginCmd() *cobra.Command{
    cmd := &cobra.Command{
        Use: "login",
        ...
    }

    cmd.Flags().StringP("username", "u", "", "the username")
    cmd.MarkFlagRequired("username")
    cmd.Flags().StringP("password", "u", "", "the password")
    cmd.MarkFlagRequired("password")

    return cmd
}

func RootCmd() *cobra.Command{
    cmd := &cobra.Command{...}

    cmd.AddCommand(
        LoginCmd(),
        UserCmd(),
        ...
    )

    return cmd
}

func main() {
    if err := RootCmd().Execute(); err != nil { ... }
}
```

Instead of having command variables, let's create command functions
that generate each command and initialise its flags, subcommands, etc.

This allows as well to run the tests in a more natural way:

```go
func TestLoginCommand(t *testing.T) {
    cmd := LoginCmd()
    cmd.SetArgs([]string{"--username", "jane_doe", "-p", "mysupersecret"})
    require.NoError(cmd.Execute())
}
```

and can be abstracted to a simple utility function that would help as
well with other lifecycle tasks like resetting the printer:

```go
func runCommand(cmd *cobra.Command, argStr string) error {
    cmd.SetArgs(strings.Split(argStr, " "))
    printer.Clean()
    return cmd.Execute()
}

func TestLoginCommand(t *testing.T) {
    err := runCommand(LoginCmd(), "--username john_doe -p mysupersecret")
    require.NoError(err)
    require.Equal(printer.GetLines()[0], "Login successful")
}
```

## Misc

 - We should follow the [XDG Base Directory
   Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
   for configuration and other files. For the case of `mmctl`, this
   means moving the configuration to the `$XDG_CONFIG_HOME` directory.
 - We should provide the user with a global flag and environment
   variable to point to a specific location for the configuration
   file.
 - We should enable a debug mode that would use the `runtime` package
   to provide with more information (namely file and line where the
   error was raised) to a savvy user.
 - This is something that we currently respect in `mmctl`, but we
   should never require the user to run a command from a specific
   location.
 - For the case of time-consuming tasks, we should show some kind of
   progress or spinner indicator, and we should allow the user to
   recover from a failure if possible.
 - All the commands should be able to run without any interaction from
   the user. This is specially important when asking for confirmation
   (we should always provide a flag/env variable to skip) and for
   "wizard" processes like the login in `mmctl`.

## Package and release strategy

Currently we have several packages for `mmctl`, each one with its own
release process, and right now we are only running the GitHub Releases
one as part of the CI. We should make sure that the packages we'd like
to support create their releases as part of the repository CI, and in
case they need manual steps to be released, we should make sure that
those steps are documented and have an owner that ensures they happen
for every release.

In the same spirit, we should modify how the GitHub releases are
currently being done, as we are creating the release tag right after
cutting the branch for the server to have a release to download, and
modifying the tag's commit every time the release branch moves
forward. A possible solution for this would be to build RC releases
that the server can use and build final just before the final server's
release.
