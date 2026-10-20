# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ice Breaker is a Flask web app (Python 3.10, Pipenv) that takes a person's name, finds their LinkedIn and Twitter profiles with LangChain ReAct agents, scrapes their profile data, and has OpenAI generate a summary, topics of interest and ice breakers.

## Commands

```bash
pipenv install                      # install deps
pipenv run python app.py            # run Flask (debug, 0.0.0.0:5000)
pipenv run black . && pipenv run isort .
pipenv run pylint <module>
```

Each integration module has an `if __name__ == "__main__":` block for manual checks. Run them as modules from the repo root so the package imports resolve:

```bash
pipenv run python -m third_parties.linkedin
pipenv run python -m third_parties.twitter
```

There is no test suite. The README mentions `pytest`, but it isn't listed in the Pipfile.

## Architecture

Request flow: `templates/index.html` POSTs the form field `name` to `/process` in `app.py`. That calls `ice_break_with(name)` in `ice_breaker.py`, which runs these steps one after another:

1. `agents/linkedin_lookup_agent.lookup`: a ReAct agent (gpt-4o-mini, prompt pulled at runtime via `hub.pull("hwchase17/react")`) that uses the Tavily search tool in `tools/tools.py` to return a LinkedIn URL.
2. `third_parties/linkedin.scrape_linkedin_profile`: Scrapin.io enrichment API. It drops empty fields and `certifications`. Pass `mock=True` to load a canned profile from a GitHub gist instead.
3. `agents/twitter_lookup_agent.lookup`: the same agent pattern, returning a Twitter username.
4. `third_parties/twitter.scrape_user_tweets_mock`: **the pipeline always uses the mock**, which fetches a fixed gist of Eden Marco's tweets no matter who was looked up. `scrape_user_tweets` (real Tweepy API) exists but isn't wired in.
5. Three separate chains in `chains/custom_chains.py` (summary, interests, ice breakers). Each is `PromptTemplate | llm | PydanticOutputParser` and gets the same `{information, twitter_posts}` input. All three use the temperature-0 gpt-3.5-turbo `llm`. `llm_creative` is defined but unused.

The Pydantic models in `output_parsers.py` (`Summary`, `TopicOfInterest`, `IceBreaker`) each have a `to_dict()` that `app.py` uses to build the JSON response (`summary_and_facts`, `interests`, `ice_breakers`, `picture_url` from LinkedIn `photoUrl`). If you change a model's fields, update `to_dict()` and the frontend JS in `index.html` as well.

## Environment / Gotchas

Modules call `load_dotenv()` themselves. The required `.env` keys are `OPENAI_API_KEY`, `SCRAPIN_API_KEY` and `TAVILY_API_KEY`.

- `third_parties/twitter.py` creates a `tweepy.Client` **when the module is imported**, using `os.environ[...]`. The app fails to start unless these are set, even though only the mock is used: `TWITTER_BEARER_TOKEN`, `TWITTER_API_KEY`, `TWITTER_API_KEY_SECRET`, `TWITTER_ACCESS_TOKEN`, `TWITTER_ACCESS_TOKEN_SECRET`. The README lists different names; the code is authoritative.
- `.env.example` is out of date: it has `PROXYCURL_API_KEY` instead of `SCRAPIN_API_KEY`.
- If `LANGCHAIN_TRACING_V2=true`, a valid `LANGCHAIN_API_KEY` is required or the app errors. Leave the tracing vars unset to disable tracing.
- The agents need network access at runtime, both for `hub.pull` and for Tavily.

## CI

`.github/workflows/claude-code-review.yml` runs automated Claude PR reviews. `claude.yml` responds to `@claude` mentions and has the Context7 MCP server configured for library docs.


# I like to eat Mamburger and Piza