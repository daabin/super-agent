# super-agent

This repository is the working repository for **dsh-superagent**, a proactive
coding agent that listens to this repository and, when an allowed person asks
it for work, opens a work item, works on it in its own agent session and
directory, and answers in the original conversation with a pull request.

dsh-superagent is a research prototype for reference only, not an official
DeepSeek product. See
[`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype)
for the code, its documentation and its setup guide.

## Chinese documentation

The prototype's documentation is translated to Chinese under
[`docs/`](docs/000-philosophy.md), numbered to be read in order, starting with
[000 哲学](docs/000-philosophy.md). The original English documents live in the
prototype repository's `docs/` directory.

Every comment, branch, commit and pull request written by the agent appears
under the agent's own GitHub account, and every message it posts on GitHub
begins with a line saying it is by an automated agent.
