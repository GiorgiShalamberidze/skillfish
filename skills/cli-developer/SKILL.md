---
name: cli-developer
description: CLI tool development: argument parsing (commander, click, cobra), interactive prompts, progress indicators, shell completions, distribution, and cross-platform.
---

# CLI Developer

Build professional, user-friendly command-line tools that handle argument parsing, interactive prompts, rich output formatting, configuration management, shell completions, error handling, testing, and distribution. Covers the three dominant CLI ecosystems: Node.js (commander/inquirer), Python (click/rich), and Go (cobra/survey) with production-ready patterns and anti-patterns to avoid.

---

## Table of Contents

- [Argument Parsing](#argument-parsing)
- [Interactive Prompts](#interactive-prompts)
- [Output and Formatting](#output-and-formatting)
- [Configuration](#configuration)
- [Shell Completions](#shell-completions)
- [Error Handling](#error-handling)
- [Testing CLIs](#testing-clis)
- [Distribution](#distribution)

---

## Argument Parsing

A well-designed CLI communicates its purpose through its flags, subcommands, and help text. Use a battle-tested parsing library rather than hand-rolling argument handling.

### Commander.js (Node.js)

```javascript
import { Command } from "commander";

const program = new Command();

program
  .name("deploy")
  .description("Deploy applications to cloud infrastructure")
  .version("1.4.0");

program
  .command("push")
  .description("Push current build to target environment")
  .argument("<environment>", "target environment (staging, production)")
  .option("-t, --tag <tag>", "Docker image tag", "latest")
  .option("-f, --force", "skip confirmation prompt", false)
  .option("--dry-run", "show what would be deployed without executing")
  .option("--timeout <seconds>", "deployment timeout", parseInt, 300)
  .action(async (environment, options) => {
    if (!["staging", "production"].includes(environment)) {
      program.error(`Invalid environment: ${environment}`);
    }
    await deploy(environment, options);
  });

program
  .command("rollback")
  .description("Rollback to a previous deployment")
  .argument("[version]", "specific version to rollback to")
  .option("--steps <n>", "rollback N deployments", parseInt, 1)
  .action(async (version, options) => {
    await rollback(version, options);
  });

program.parse();
```

### Click (Python)

```python
import click

@click.group()
@click.version_option(version="1.4.0")
@click.option("--verbose", "-v", count=True, help="Increase verbosity (-v, -vv, -vvv)")
@click.pass_context
def cli(ctx, verbose):
    """Deploy applications to cloud infrastructure."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose

@cli.command()
@click.argument("environment", type=click.Choice(["staging", "production"]))
@click.option("--tag", "-t", default="latest", help="Docker image tag")
@click.option("--force", "-f", is_flag=True, help="Skip confirmation prompt")
@click.option("--dry-run", is_flag=True, help="Show plan without executing")
@click.option("--timeout", default=300, type=int, help="Deployment timeout in seconds")
@click.pass_context
def push(ctx, environment, tag, force, dry_run, timeout):
    """Push current build to target environment."""
    verbose = ctx.obj["verbose"]
    deploy(environment, tag=tag, force=force, dry_run=dry_run,
           timeout=timeout, verbose=verbose)

@cli.command()
@click.argument("version", required=False)
@click.option("--steps", default=1, type=int, help="Rollback N deployments")
@click.pass_context
def rollback(ctx, version, steps):
    """Rollback to a previous deployment."""
    perform_rollback(version, steps=steps)

if __name__ == "__main__":
    cli()
```

### Cobra (Go)

```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var (
    tag     string
    force   bool
    dryRun  bool
    timeout int
)

var rootCmd = &cobra.Command{
    Use:     "deploy",
    Short:   "Deploy applications to cloud infrastructure",
    Version: "1.4.0",
}

var pushCmd = &cobra.Command{
    Use:   "push <environment>",
    Short: "Push current build to target environment",
    Args:  cobra.ExactArgs(1),
    ValidArgs: []string{"staging", "production"},
    RunE: func(cmd *cobra.Command, args []string) error {
        env := args[0]
        if env != "staging" && env != "production" {
            return fmt.Errorf("invalid environment: %s", env)
        }
        return runDeploy(env, tag, force, dryRun, timeout)
    },
}

var rollbackCmd = &cobra.Command{
    Use:   "rollback [version]",
    Short: "Rollback to a previous deployment",
    Args:  cobra.MaximumNArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        version := ""
        if len(args) > 0 {
            version = args[0]
        }
        steps, _ := cmd.Flags().GetInt("steps")
        return runRollback(version, steps)
    },
}

func init() {
    pushCmd.Flags().StringVarP(&tag, "tag", "t", "latest", "Docker image tag")
    pushCmd.Flags().BoolVarP(&force, "force", "f", false, "Skip confirmation")
    pushCmd.Flags().BoolVar(&dryRun, "dry-run", false, "Show plan without executing")
    pushCmd.Flags().IntVar(&timeout, "timeout", 300, "Deployment timeout in seconds")

    rollbackCmd.Flags().Int("steps", 1, "Rollback N deployments")

    rootCmd.AddCommand(pushCmd, rollbackCmd)
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

### Anti-Patterns

- **Parsing `process.argv` / `sys.argv` / `os.Args` by hand** -- You will miss edge cases (quoted strings, `=` in values, combined short flags). Use a library.
- **Positional arguments for everything** -- More than two positional args becomes confusing. Use named flags.
- **No help text** -- Every command and flag needs a description. Auto-generated help is a primary user interface.
- **Mixing global and local flags** -- Global flags go on the root command. Subcommand-specific flags go on the subcommand.

---

## Interactive Prompts

Interactive prompts guide users through complex input. Always provide a non-interactive path (flags/env vars) so the CLI works in CI pipelines.

### Inquirer / @inquirer/prompts (Node.js)

```javascript
import { input, select, confirm, checkbox } from "@inquirer/prompts";

async function setupProject() {
  const name = await input({
    message: "Project name:",
    default: "my-app",
    validate: (value) =>
      /^[a-z0-9-]+$/.test(value) || "Lowercase letters, numbers, and hyphens only",
  });

  const framework = await select({
    message: "Framework:",
    choices: [
      { name: "Express", value: "express", description: "Minimal and flexible" },
      { name: "Fastify", value: "fastify", description: "High performance" },
      { name: "Hono", value: "hono", description: "Ultrafast, runs everywhere" },
    ],
  });

  const features = await checkbox({
    message: "Features to include:",
    choices: [
      { name: "TypeScript", value: "typescript", checked: true },
      { name: "ESLint + Prettier", value: "linting" },
      { name: "Docker", value: "docker" },
      { name: "GitHub Actions CI", value: "ci" },
    ],
  });

  const proceed = await confirm({
    message: `Create ${name} with ${framework}?`,
    default: true,
  });

  if (proceed) {
    await scaffoldProject({ name, framework, features });
  }
}
```

### Questionary + Rich (Python)

```python
import questionary
from rich.console import Console

console = Console()

def setup_project():
    name = questionary.text(
        "Project name:",
        default="my-app",
        validate=lambda v: bool(v.strip()) or "Name cannot be empty",
    ).ask()

    framework = questionary.select(
        "Framework:",
        choices=["Flask", "FastAPI", "Django"],
    ).ask()

    features = questionary.checkbox(
        "Features to include:",
        choices=[
            questionary.Choice("Type hints (mypy)", checked=True),
            questionary.Choice("Ruff linting"),
            questionary.Choice("Docker"),
            questionary.Choice("GitHub Actions CI"),
        ],
    ).ask()

    if questionary.confirm(f"Create {name} with {framework}?", default=True).ask():
        scaffold_project(name, framework, features)
```

### Survey (Go)

```go
package main

import "github.com/AlecAivazis/survey/v2"

type ProjectConfig struct {
    Name      string
    Framework string
    Features  []string
    Confirm   bool
}

func setupProject() (*ProjectConfig, error) {
    cfg := &ProjectConfig{}

    questions := []*survey.Question{
        {
            Name: "name",
            Prompt: &survey.Input{
                Message: "Project name:",
                Default: "my-app",
            },
            Validate: survey.Required,
        },
        {
            Name: "framework",
            Prompt: &survey.Select{
                Message: "Framework:",
                Options: []string{"Gin", "Echo", "Fiber"},
                Default: "Gin",
            },
        },
        {
            Name: "features",
            Prompt: &survey.MultiSelect{
                Message: "Features to include:",
                Options: []string{"golangci-lint", "Docker", "GitHub Actions CI", "Air (hot reload)"},
                Default: []string{"golangci-lint"},
            },
        },
        {
            Name: "confirm",
            Prompt: &survey.Confirm{
                Message: "Proceed with scaffolding?",
                Default: true,
            },
        },
    }

    if err := survey.Ask(questions, cfg); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

### Anti-Patterns

- **Prompts with no non-interactive alternative** -- CI environments cannot answer prompts. Accept `--yes` / `--name` flags that skip interactive mode.
- **No default values** -- Users should be able to press Enter through common cases.
- **Long prompt chains with no escape** -- Let users Ctrl+C cleanly. Handle `nil`/empty returns from cancelled prompts.
- **Prompting for values already provided as flags** -- Check flags first, only prompt for missing values.

---

## Output and Formatting

Good CLI output adapts: color and formatting for terminals, plain text or JSON for pipes. Detect whether stdout is a TTY and adjust accordingly.

### Chalk + ora (Node.js)

```javascript
import chalk from "chalk";
import ora from "ora";
import { table } from "table";

function isInteractive() {
  return process.stdout.isTTY && !process.env.CI;
}

function formatStatus(status) {
  const colors = { running: chalk.green, stopped: chalk.red, pending: chalk.yellow };
  return (colors[status] || chalk.white)(status);
}

async function deployWithProgress(services) {
  if (process.env.JSON_OUTPUT) {
    // Machine-readable output for piping
    const results = await deployAll(services);
    console.log(JSON.stringify(results, null, 2));
    return;
  }

  const spinner = ora("Deploying services...").start();

  for (const service of services) {
    spinner.text = `Deploying ${chalk.bold(service.name)}...`;
    await deploy(service);
    spinner.succeed(`${chalk.bold(service.name)} deployed`);
    spinner = ora().start();
  }

  spinner.stop();

  // Summary table
  const data = [
    ["Service", "Status", "URL"],
    ...services.map((s) => [
      chalk.bold(s.name),
      formatStatus(s.status),
      chalk.underline(s.url),
    ]),
  ];
  console.log(table(data));
}
```

### Rich (Python)

```python
import json
import sys
from rich.console import Console
from rich.table import Table
from rich.progress import Progress, SpinnerColumn, TextColumn, BarColumn
from rich.panel import Panel

console = Console()
error_console = Console(stderr=True)

def deploy_with_progress(services, json_output=False):
    if json_output:
        results = deploy_all(services)
        print(json.dumps(results, indent=2))
        return

    with Progress(
        SpinnerColumn(),
        TextColumn("[bold]{task.description}"),
        BarColumn(),
        TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
        console=console,
    ) as progress:
        task = progress.add_task("Deploying...", total=len(services))
        for service in services:
            progress.update(task, description=f"Deploying {service['name']}...")
            deploy(service)
            progress.advance(task)

    # Summary table
    tbl = Table(title="Deployment Summary")
    tbl.add_column("Service", style="bold")
    tbl.add_column("Status")
    tbl.add_column("URL", style="underline blue")

    for s in services:
        status_style = {"running": "green", "stopped": "red", "pending": "yellow"}
        tbl.add_row(s["name"], f"[{status_style.get(s['status'], 'white')}]{s['status']}", s["url"])

    console.print(tbl)
```

### Lipgloss + bubbletea (Go)

```go
package main

import (
    "fmt"
    "os"

    "github.com/charmbracelet/lipgloss"
)

var (
    titleStyle  = lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("12"))
    successStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("10"))
    errorStyle   = lipgloss.NewStyle().Foreground(lipgloss.Color("9"))
    urlStyle     = lipgloss.NewStyle().Underline(true).Foreground(lipgloss.Color("12"))
)

func printSummary(services []Service) {
    if os.Getenv("JSON_OUTPUT") != "" {
        printJSON(services)
        return
    }

    fmt.Println(titleStyle.Render("Deployment Summary"))
    fmt.Println()

    for _, s := range services {
        status := successStyle.Render("running")
        if s.Status == "stopped" {
            status = errorStyle.Render("stopped")
        }
        fmt.Printf("  %-20s %s  %s\n", s.Name, status, urlStyle.Render(s.URL))
    }
}
```

### Verbose and Quiet Flags

```javascript
// Standard verbosity pattern: -q (quiet), default (normal), -v (verbose), -vv (debug)
function createLogger(verbosity) {
  return {
    debug: (msg) => verbosity >= 2 && console.error("[debug]", msg),
    info: (msg) => verbosity >= 1 && console.error(msg),
    warn: (msg) => verbosity >= 0 && console.error(chalk.yellow("warning:"), msg),
    error: (msg) => console.error(chalk.red("error:"), msg),
    // Data output always goes to stdout (never stderr) so it can be piped
    output: (data) => console.log(data),
  };
}
```

### Anti-Patterns

- **Mixing data and status messages on stdout** -- Status/progress goes to stderr, data goes to stdout. This lets `mycli list | jq .` work.
- **Colors with no TTY check** -- Piped output should be plain. Libraries like chalk auto-detect, but verify.
- **No `--json` flag** -- Machine-readable output is essential for scripting.
- **Giant walls of text** -- Use tables, sections, and whitespace. Highlight what matters.

---

## Configuration

CLI tools need layered configuration: defaults < config file < environment variables < flags. Discover config files using established conventions.

### Config File Discovery (Node.js)

```javascript
import { readFile } from "node:fs/promises";
import { resolve, dirname } from "node:path";
import { homedir } from "node:os";
import { parse as parseToml } from "@iarna/toml";

const CONFIG_FILES = [".deployrc.toml", ".deployrc.json", "deploy.config.js"];

async function findConfig(startDir = process.cwd()) {
  // Walk up from cwd to find nearest config file
  let dir = startDir;
  while (true) {
    for (const filename of CONFIG_FILES) {
      const filepath = resolve(dir, filename);
      try {
        const content = await readFile(filepath, "utf-8");
        return { filepath, content, format: filename.split(".").pop() };
      } catch {}
    }
    const parent = dirname(dir);
    if (parent === dir) break;
    dir = parent;
  }

  // Fall back to ~/.config/deploy/config.toml (XDG convention)
  const xdgConfig = resolve(
    process.env.XDG_CONFIG_HOME || resolve(homedir(), ".config"),
    "deploy",
    "config.toml"
  );
  try {
    const content = await readFile(xdgConfig, "utf-8");
    return { filepath: xdgConfig, content, format: "toml" };
  } catch {
    return null;
  }
}
```

### Layered Config with Validation (Python)

```python
import os
from pathlib import Path
from pydantic import BaseModel, Field
import tomllib

class DeployConfig(BaseModel):
    environment: str = "staging"
    tag: str = "latest"
    timeout: int = Field(default=300, ge=10, le=3600)
    registry: str = "ghcr.io"
    notify_slack: bool = False

def load_config(cli_overrides: dict) -> DeployConfig:
    """Load config with precedence: CLI flags > env vars > config file > defaults."""
    # 1. Config file
    file_config = {}
    config_path = Path.home() / ".config" / "deploy" / "config.toml"
    project_config = Path.cwd() / ".deployrc.toml"

    for path in [config_path, project_config]:  # project overrides global
        if path.exists():
            with open(path, "rb") as f:
                file_config.update(tomllib.load(f))

    # 2. Environment variables (DEPLOY_ prefix)
    env_config = {}
    prefix = "DEPLOY_"
    for key in DeployConfig.model_fields:
        env_key = f"{prefix}{key.upper()}"
        if env_key in os.environ:
            env_config[key] = os.environ[env_key]

    # 3. Merge: defaults < file < env < CLI
    merged = {**file_config, **env_config, **{k: v for k, v in cli_overrides.items() if v is not None}}
    return DeployConfig(**merged)
```

### Viper (Go)

```go
package config

import (
    "fmt"
    "strings"

    "github.com/spf13/viper"
)

type Config struct {
    Environment string `mapstructure:"environment"`
    Tag         string `mapstructure:"tag"`
    Timeout     int    `mapstructure:"timeout"`
    Registry    string `mapstructure:"registry"`
    NotifySlack bool   `mapstructure:"notify_slack"`
}

func Load() (*Config, error) {
    v := viper.New()

    // Defaults
    v.SetDefault("environment", "staging")
    v.SetDefault("tag", "latest")
    v.SetDefault("timeout", 300)
    v.SetDefault("registry", "ghcr.io")

    // Config file discovery
    v.SetConfigName(".deployrc")
    v.SetConfigType("toml")
    v.AddConfigPath(".")           // Project directory first
    v.AddConfigPath("$HOME/.config/deploy") // XDG fallback

    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("config error: %w", err)
        }
    }

    // Environment variables with DEPLOY_ prefix
    v.SetEnvPrefix("DEPLOY")
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    v.AutomaticEnv()

    cfg := &Config{}
    if err := v.Unmarshal(cfg); err != nil {
        return nil, fmt.Errorf("config unmarshal error: %w", err)
    }
    return cfg, nil
}
```

### Anti-Patterns

- **Inventing a new config format** -- Use TOML, JSON, or YAML. Never invent custom syntax.
- **No config file validation** -- Typos in config keys should produce warnings, not silent failures.
- **Hardcoded config paths** -- Respect XDG_CONFIG_HOME on Linux, and allow `--config` to override.
- **Config file required to run** -- Every value should have a sensible default. The tool should work with zero config for simple cases.

---

## Shell Completions

Shell completions make a CLI feel native. Generate them at install time or provide a `completion` subcommand.

### Commander.js Completions (Node.js)

```javascript
import { Command } from "commander";

// Built-in: commander does not auto-generate completions.
// Use tabtab or omelette for Node.js CLIs.

import { default as tabtab } from "tabtab";

async function installCompletions() {
  await tabtab.install({
    name: "deploy",
    completer: "deploy",
  });
}

async function handleCompletions() {
  const env = tabtab.parseEnv(process.env);
  if (!env.complete) return false;

  if (env.prev === "deploy") {
    await tabtab.log(["push", "rollback", "status", "completion"]);
  } else if (env.prev === "push") {
    await tabtab.log(["staging", "production"]);
  } else if (env.prev === "--tag" || env.prev === "-t") {
    // Dynamic: fetch recent tags from registry
    const tags = await fetchRecentTags();
    await tabtab.log(tags);
  }
  return true;
}
```

### Click Completions (Python)

```python
# Click has built-in completion support for bash, zsh, and fish.
# Users activate it with an environment variable:

# bash:  eval "$(_DEPLOY_COMPLETE=bash_source deploy)"
# zsh:   eval "$(_DEPLOY_COMPLETE=zsh_source deploy)"
# fish:  _DEPLOY_COMPLETE=fish_source deploy | source

# Provide a completion subcommand for easy setup:
import click
import os
import sys

@cli.command()
@click.argument("shell", type=click.Choice(["bash", "zsh", "fish"]))
def completion(shell):
    """Generate shell completion script."""
    env_var = "_DEPLOY_COMPLETE"
    source_type = f"{shell}_source"
    os.environ[env_var] = source_type
    # Re-import to trigger completion output
    sys.exit(0)

# For custom dynamic completions:
def get_environments(ctx, args, incomplete):
    envs = ["staging", "production", "development"]
    return [e for e in envs if e.startswith(incomplete)]

@cli.command()
@click.argument("environment", type=click.STRING, shell_complete=get_environments)
def push(environment):
    """Push to environment with tab completion."""
    pass
```

### Cobra Completions (Go)

```go
// Cobra has first-class completion support built in.

// Register dynamic completions on a command:
var pushCmd = &cobra.Command{
    Use:   "push <environment>",
    Short: "Push to target environment",
    // Static valid args
    ValidArgs: []string{"staging", "production"},
    // Or dynamic completion function:
    ValidArgsFunction: func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
        if len(args) != 0 {
            return nil, cobra.ShellCompDirectiveNoFileComp
        }
        return getEnvironments(toComplete), cobra.ShellCompDirectiveNoFileComp
    },
    RunE: pushRun,
}

// Register flag completions:
func init() {
    pushCmd.Flags().StringVarP(&tag, "tag", "t", "latest", "Image tag")
    pushCmd.RegisterFlagCompletionFunc("tag", func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
        tags, _ := fetchRecentTags()
        return tags, cobra.ShellCompDirectiveNoFileComp
    })

    // Cobra auto-generates the `completion` subcommand:
    //   deploy completion bash
    //   deploy completion zsh
    //   deploy completion fish
    //   deploy completion powershell
}
```

### Anti-Patterns

- **No completions at all** -- Users expect tab completion. At minimum, support subcommand and flag name completion.
- **Only static completions** -- Dynamic completions (fetching tags, listing environments) dramatically improve UX.
- **Completion installation requires manual shell editing** -- Provide a `completion --install` command or print eval-able output.

---

## Error Handling

A CLI's error handling determines whether users can self-serve or file support tickets. Use exit codes, structured error messages, and debug modes.

### Exit Codes Convention

```
0   Success
1   General error
2   Usage error (bad arguments, missing required flags)
64  Command line usage error (EX_USAGE from sysexits.h)
65  Data format error
69  Unavailable service (network, API down)
70  Internal software error
130 Interrupted (Ctrl+C / SIGINT)
```

### Structured Error Handling (Node.js)

```javascript
class CliError extends Error {
  constructor(message, { exitCode = 1, hint, cause } = {}) {
    super(message);
    this.exitCode = exitCode;
    this.hint = hint;
    this.cause = cause;
  }
}

function handleError(error) {
  if (error instanceof CliError) {
    console.error(`${chalk.red("error:")} ${error.message}`);
    if (error.hint) {
      console.error(`${chalk.yellow("hint:")} ${error.hint}`);
    }
    if (process.env.DEBUG) {
      console.error("\nStack trace:");
      console.error(error.stack);
      if (error.cause) {
        console.error("\nCaused by:", error.cause);
      }
    }
    process.exit(error.exitCode);
  }

  // Unexpected error
  console.error(`${chalk.red("unexpected error:")} ${error.message}`);
  console.error(`\nThis is a bug. Please report it at: https://github.com/org/deploy/issues`);
  if (process.env.DEBUG) {
    console.error(error.stack);
  }
  process.exit(70);
}

// Usage
function requireAuth() {
  if (!process.env.DEPLOY_TOKEN) {
    throw new CliError("Not authenticated", {
      exitCode: 2,
      hint: 'Run "deploy login" or set DEPLOY_TOKEN environment variable',
    });
  }
}

process.on("uncaughtException", handleError);
process.on("unhandledRejection", handleError);
```

### Click Error Handling (Python)

```python
import click
import sys
import traceback

class CliError(click.ClickException):
    def __init__(self, message, hint=None, exit_code=1):
        super().__init__(message)
        self.exit_code = exit_code
        self.hint = hint

    def format_message(self):
        msg = f"Error: {self.message}"
        if self.hint:
            msg += f"\nHint: {self.hint}"
        return msg

def error_handler(func):
    """Decorator that wraps commands with consistent error handling."""
    @click.pass_context
    def wrapper(ctx, *args, **kwargs):
        try:
            return ctx.invoke(func, *args, **kwargs)
        except CliError:
            raise  # Let Click handle CliError formatting
        except ConnectionError as e:
            raise CliError(
                f"Cannot reach API: {e}",
                hint="Check your network connection and try again",
                exit_code=69,
            )
        except Exception as e:
            if ctx.obj.get("verbose", 0) >= 2:
                traceback.print_exc()
            raise CliError(
                f"Unexpected error: {e}",
                hint="Run with -vv for full traceback, or file a bug report",
                exit_code=70,
            )
    return wrapper
```

### Cobra Error Handling (Go)

```go
package cmd

import (
    "errors"
    "fmt"
    "os"
)

type CliError struct {
    Message  string
    Hint     string
    ExitCode int
    Cause    error
}

func (e *CliError) Error() string { return e.Message }
func (e *CliError) Unwrap() error { return e.Cause }

func handleError(err error) {
    if err == nil {
        return
    }

    var cliErr *CliError
    if errors.As(err, &cliErr) {
        fmt.Fprintf(os.Stderr, "error: %s\n", cliErr.Message)
        if cliErr.Hint != "" {
            fmt.Fprintf(os.Stderr, "hint: %s\n", cliErr.Hint)
        }
        if os.Getenv("DEBUG") != "" && cliErr.Cause != nil {
            fmt.Fprintf(os.Stderr, "\ncause: %+v\n", cliErr.Cause)
        }
        os.Exit(cliErr.ExitCode)
    }

    fmt.Fprintf(os.Stderr, "unexpected error: %s\n", err)
    fmt.Fprintf(os.Stderr, "Please report this bug.\n")
    os.Exit(70)
}
```

### Anti-Patterns

- **Printing stack traces by default** -- Stack traces are for developers, not users. Hide behind `--debug` or `DEBUG=1`.
- **Exit code 1 for everything** -- Use distinct codes so scripts can distinguish "bad arguments" from "network failure."
- **Error messages with no next step** -- Always include a hint: what to check, what command to run, or where to get help.
- **Swallowing errors silently** -- A non-zero exit with no output is worse than a crash. Always print something.

---

## Testing CLIs

CLI testing requires verifying both the output text and exit codes. Test at two levels: unit-test the business logic, integration-test the full CLI process.

### Snapshot Testing (Node.js with Vitest)

```javascript
import { describe, it, expect } from "vitest";
import { execFileSync } from "node:child_process";

function runCli(args, { env = {}, input } = {}) {
  try {
    const stdout = execFileSync("node", ["./bin/deploy.js", ...args], {
      encoding: "utf-8",
      env: { ...process.env, ...env, NO_COLOR: "1" },
      input,
      timeout: 10_000,
    });
    return { stdout, exitCode: 0 };
  } catch (error) {
    return { stdout: error.stdout, stderr: error.stderr, exitCode: error.status };
  }
}

describe("deploy CLI", () => {
  it("shows help text", () => {
    const result = runCli(["--help"]);
    expect(result.exitCode).toBe(0);
    expect(result.stdout).toMatchSnapshot();
  });

  it("rejects invalid environment", () => {
    const result = runCli(["push", "invalid-env"]);
    expect(result.exitCode).toBe(1);
    expect(result.stderr).toContain("Invalid environment");
  });

  it("outputs JSON when requested", () => {
    const result = runCli(["status"], { env: { JSON_OUTPUT: "1" } });
    expect(result.exitCode).toBe(0);
    const data = JSON.parse(result.stdout);
    expect(data).toHaveProperty("services");
  });
});
```

### Click Testing (Python)

```python
from click.testing import CliRunner
from deploy.cli import cli

def test_help_text():
    runner = CliRunner()
    result = runner.invoke(cli, ["--help"])
    assert result.exit_code == 0
    assert "Deploy applications" in result.output

def test_push_requires_environment():
    runner = CliRunner()
    result = runner.invoke(cli, ["push"])
    assert result.exit_code != 0
    assert "Missing argument" in result.output

def test_push_dry_run(mocker):
    mock_deploy = mocker.patch("deploy.core.deploy")
    runner = CliRunner()
    result = runner.invoke(cli, ["push", "staging", "--dry-run"])
    assert result.exit_code == 0
    assert "DRY RUN" in result.output
    mock_deploy.assert_not_called()

def test_config_from_env():
    runner = CliRunner(env={"DEPLOY_TAG": "v2.0"})
    result = runner.invoke(cli, ["push", "staging", "--dry-run"])
    assert result.exit_code == 0
    assert "v2.0" in result.output

def test_interactive_prompt():
    runner = CliRunner()
    result = runner.invoke(cli, ["init"], input="my-project\n1\ny\n")
    assert result.exit_code == 0
    assert "my-project" in result.output
```

### Go CLI Testing

```go
package cmd_test

import (
    "bytes"
    "os/exec"
    "strings"
    "testing"
)

func runCli(t *testing.T, args ...string) (string, string, int) {
    t.Helper()
    cmd := exec.Command("./deploy", args...)
    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr
    err := cmd.Run()
    exitCode := 0
    if exitErr, ok := err.(*exec.ExitError); ok {
        exitCode = exitErr.ExitCode()
    }
    return stdout.String(), stderr.String(), exitCode
}

func TestHelpText(t *testing.T) {
    stdout, _, code := runCli(t, "--help")
    if code != 0 {
        t.Fatalf("expected exit 0, got %d", code)
    }
    if !strings.Contains(stdout, "Deploy applications") {
        t.Error("help text missing description")
    }
}

func TestPushInvalidEnvironment(t *testing.T) {
    _, stderr, code := runCli(t, "push", "invalid")
    if code == 0 {
        t.Fatal("expected non-zero exit code")
    }
    if !strings.Contains(stderr, "invalid environment") {
        t.Errorf("expected error message, got: %s", stderr)
    }
}

// Unit-test the business logic separately from the CLI layer
func TestDeployLogic(t *testing.T) {
    cfg := &DeployConfig{Environment: "staging", Tag: "v1.0", DryRun: true}
    result, err := deploy(cfg)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result.Status != "planned" {
        t.Errorf("expected planned status for dry run, got %s", result.Status)
    }
}
```

### Anti-Patterns

- **Only testing the library functions, not the CLI itself** -- The argument parsing, flag defaults, and output formatting are all part of the contract. Test the full binary.
- **Testing with colors enabled** -- Set `NO_COLOR=1` or equivalent. Color codes break string assertions and snapshot tests.
- **No timeout on subprocess tests** -- A hanging CLI test blocks the entire CI suite. Always set a timeout.
- **Ignoring exit codes in tests** -- A test that only checks stdout but ignores a non-zero exit code is incomplete.

---

## Distribution

Ship your CLI so users can install it with a single command, regardless of their platform or package manager.

### npm Publish (Node.js)

```json
{
  "name": "@myorg/deploy-cli",
  "version": "1.4.0",
  "description": "Deploy applications to cloud infrastructure",
  "bin": {
    "deploy": "./bin/deploy.js"
  },
  "files": ["bin/", "src/", "LICENSE"],
  "engines": {
    "node": ">=18"
  },
  "scripts": {
    "prepare": "npm run build",
    "test": "vitest run"
  }
}
```

```bash
# bin/deploy.js must start with shebang:
#!/usr/bin/env node

# Publish to npm
npm publish --access public

# Users install globally:
npm install -g @myorg/deploy-cli
# Or run without installing:
npx @myorg/deploy-cli push staging
```

### PyPI Distribution (Python)

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "deploy-cli"
version = "1.4.0"
description = "Deploy applications to cloud infrastructure"
requires-python = ">=3.10"
dependencies = ["click>=8.0", "rich>=13.0", "pydantic>=2.0"]

[project.scripts]
deploy = "deploy_cli.cli:cli"

[tool.hatch.build.targets.wheel]
packages = ["src/deploy_cli"]
```

```bash
# Build and upload to PyPI
python -m build
twine upload dist/*

# Users install with pip or pipx (isolated):
pipx install deploy-cli
```

### Go Cross-Compilation with GoReleaser

```yaml
# .goreleaser.yaml
version: 2
builds:
  - main: ./cmd/deploy
    binary: deploy
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w -X main.version={{.Version}}

archives:
  - formats: ["tar.gz"]
    format_overrides:
      - goos: windows
        formats: ["zip"]
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"

brews:
  - repository:
      owner: myorg
      name: homebrew-tap
    homepage: "https://github.com/myorg/deploy"
    description: "Deploy applications to cloud infrastructure"
    install: |
      bin.install "deploy"

checksum:
  name_template: "checksums.txt"

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
```

```bash
# Tag and release
git tag v1.4.0
git push origin v1.4.0
# GoReleaser runs in CI (GitHub Actions) and uploads binaries + Homebrew formula

# Users install via Homebrew:
brew install myorg/tap/deploy
# Or download binary directly from GitHub Releases
```

### Standalone Binaries for Node.js (pkg / bun)

```bash
# Using Bun to compile a standalone binary
bun build ./bin/deploy.js --compile --outfile deploy
# Produces a single executable with Bun runtime embedded

# Using pkg (for Node.js)
npx pkg . --targets node18-linux-x64,node18-macos-x64,node18-win-x64 --output deploy
```

### Auto-Update Mechanism

```javascript
import { execSync } from "node:child_process";
import semver from "semver";

async function checkForUpdate(currentVersion) {
  try {
    const response = await fetch(
      "https://registry.npmjs.org/@myorg/deploy-cli/latest"
    );
    const { version: latest } = await response.json();

    if (semver.gt(latest, currentVersion)) {
      console.error(
        `\nUpdate available: ${currentVersion} -> ${latest}` +
        `\nRun: npm install -g @myorg/deploy-cli`
      );
    }
  } catch {
    // Never block the CLI for an update check failure
  }
}
```

```python
# Python: use a background check with cache
import json
import time
from pathlib import Path

CACHE_FILE = Path.home() / ".cache" / "deploy" / "update-check.json"
CHECK_INTERVAL = 86400  # 24 hours

def check_for_update(current_version: str):
    try:
        cache = json.loads(CACHE_FILE.read_text()) if CACHE_FILE.exists() else {}
        if time.time() - cache.get("last_check", 0) < CHECK_INTERVAL:
            return

        from importlib.metadata import version as get_version
        import urllib.request
        resp = urllib.request.urlopen("https://pypi.org/pypi/deploy-cli/json")
        latest = json.loads(resp.read())["info"]["version"]

        CACHE_FILE.parent.mkdir(parents=True, exist_ok=True)
        CACHE_FILE.write_text(json.dumps({"last_check": time.time(), "latest": latest}))

        if latest != current_version:
            import sys
            print(
                f"\nUpdate available: {current_version} -> {latest}"
                f"\nRun: pipx upgrade deploy-cli",
                file=sys.stderr,
            )
    except Exception:
        pass  # Never block the CLI for an update check
```

### Anti-Patterns

- **Requiring a runtime to use the CLI** -- Users should not need to install Node.js/Python/Go to use your tool. Offer standalone binaries for broad reach.
- **No versioning in the binary** -- Embed the version at build time. `--version` should always work and match the release.
- **Update checks that block startup** -- Run update checks asynchronously or cache results. Never add latency to every invocation.
- **No checksum verification** -- Published binaries should include SHA-256 checksums and ideally GPG signatures.
- **Forgetting Windows** -- Test on Windows. Path separators, shell differences, and missing POSIX tools are common failure points.
