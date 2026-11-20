# OpenLinc

Build your own programs on live Somnolinc Headband data, together with a coding agent.

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/). It fetches Python and
   everything else on first use.
2. Open your coding agent in this folder.
3. Say what you want to build, for example "plot my live EEG" or "make a game I steer by tilting
   my head".

The agent starts the bridge, shows you a QR code to scan with the Somnolinc app, and writes your
program next to this file.

The agent's instructions are in `AGENTS.md` and `skills/openlinc/SKILL.md`. If your agent does not
read `AGENTS.md` by itself, point it to that file first.
