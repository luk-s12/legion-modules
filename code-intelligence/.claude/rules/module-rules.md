### allow-codegraph-install
Installing CodeGraph requires an available `npm` executable. Only after this rule was accepted and
only when the CodeGraph CLI is unavailable may an agent using `code-intelligence` first run
`npm --version`; only when that check succeeds may it install the exact
`@colbymchenry/codegraph@1.5.0` version globally from the npm registry once, verify it, and continue.
If npm is unavailable or that preflight fails, the agent must report `npm_unavailable`, fall back to
conventional search, and not attempt installation. If this rule was not accepted, the agent must
use conventional search without checking npm, attempting installation, or stopping the story.
