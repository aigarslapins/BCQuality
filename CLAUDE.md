# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

BCQuality is a curated knowledge base and skills library for Business Central (BC) development. It provides structured, machine-readable guidance that LLM-driven review agents consume — establishing a consistent quality bar across teams. It contains **content** (knowledge files and skills); agents that consume BCQuality live elsewhere (AL-Go, VS Code extensions, etc.).

The admission test for any knowledge file: *"If this file did not exist, would a modern LLM reviewing or generating BC code make a mistake this file would have prevented?"* Generic software-engineering advice does not belong here regardless of how sound it is.

## Further reading

Read [agent-consumption.md](agent-consumption.md) for how agents consume this repository — structure, formats, CI rules, and agent flow architecture.
