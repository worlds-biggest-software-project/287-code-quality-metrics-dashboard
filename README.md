# Code Quality Metrics Dashboard

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source dashboard for tracking maintainability, complexity, duplication, and quality trends across modern codebases.

The Code Quality Metrics Dashboard measures and visualises code health — cyclomatic complexity, duplication, test coverage, code smells, and security issues — and tracks how those metrics evolve over time. It is built for engineering managers, platform teams, and VPs of Engineering who need defensible, trend-based evidence of code health rather than point-in-time snapshots.

---

## Why Code Quality Metrics Dashboard?

- SonarQube dominates the space (400K+ organisations, 30+ languages) but Enterprise pricing starts at $20K+/yr with unpredictable LOC-based billing, pushing mid-market teams toward alternatives.
- Mid-market alternatives (Codacy at $15/user/mo, Qodana at $6/contributor/mo, CodeAnt AI at $24–$40/user/mo) trade cost for narrower analytics, smaller rule libraries, or ecosystem bias.
- 70%+ of professional developers now use AI coding tools weekly, and 48% of engineering leaders report that code quality has become harder to maintain — creating urgency for automated quality gates that handle AI-generated code.
- Buyers increasingly want unified dashboards combining quality, security, and supply-chain metrics rather than three disconnected tools.
- Trend tracking — metrics over time, not just at a single commit — is becoming the primary buying requirement, and many incumbents still treat it as a secondary feature.

---

## Key Features

### Core Quality Measurement

- Cyclomatic complexity calculation (McCabe), with high-risk thresholds
- Code duplication detection
- Test coverage measurement
- Basic code smell detection
- Security vulnerability scanning (SAST overlap)

### Trends, Dashboards, and Workflow

- Historical trend tracking across commits, sprints, and releases
- Dashboard and visualization for team, project, and repo views
- CI/CD integration for quality gates in build pipelines
- Issue tracking and assignment for surfaced findings

### Advanced Analytics (v1.1)

- Machine learning anomaly detection on quality metrics
- Automatic issue triage and categorization
- Technical debt estimation and tracking
- Code hotspot identification
- Performance anti-pattern detection
- Dependency vulnerability scanning
- Custom rule engine
- Team metrics and contribution tracking

### Backlog Capabilities

- Architectural pattern analysis
- Machine learning bug prediction
- Cross-repository benchmarking
- Automatic fix suggestion
- Developer education and training
- Integration with code review workflow
- Language-specific best-practice detection
- Regulatory compliance checking

---

## AI-Native Advantage

AI is used to translate raw metrics into action: plain-English explanations of why a file is high-complexity (contextualised by purpose and change history), automatic prioritisation of the technical-debt backlog by correlating quality scores with incident history and bug density, and predictions of which refactors will most improve the maintainability index. The dashboard also flags AI-generated code patterns that statistically correlate with future defects — catching issues introduced by coding assistants before they reach production — and produces executive-ready quality trend summaries without manual report compilation.

---

## Tech Stack & Deployment

The project aligns with established quality standards: cyclomatic complexity (McCabe), the Maintainability Index, ISO/IEC 25010 (SQuaRE), and DORA metrics (where code quality influences change failure rate and MTTR). It overlaps with SAST disciplines, mirroring how SonarQube, Semgrep, and Snyk Code combine quality and security in one scan. Deployment modes and SDK details will be defined during specification.

---

## Market Context

The global static analysis / code quality tools market is estimated at USD 1.8 billion in 2025, growing at ~22% CAGR, accelerated by AI-generated code concerns. SonarSource has raised $412M at a $4.7B valuation; Semgrep $53M Series C; DeepSource $12M Series A. Primary buyers are engineering managers tracking technical debt, VPs of Engineering reporting code health to leadership, platform teams enforcing CI/CD quality gates, and security engineers requiring SAST alongside quality metrics.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
