# Real Estate Investment Analysis

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for underwriting real estate deals, tracking portfolio performance, and generating investor-ready reports — built for the mid-market gap between $20/month prosumer tools and enterprise platforms costing tens of thousands per year.

Real Estate Investment Analysis provides DCF modelling, market comps, ROI projections, and deal tracking for syndicators, private equity real estate funds, family offices, and individual investors. It targets the underserved space between consumer apps like DealCheck and institutional incumbents like ARGUS Enterprise, where most operators today fall back to Excel spreadsheets.

---

## Why Real Estate Investment Analysis?

- **The mid-market is unserved.** Tools below $100/month (DealCheck, Stessa, PropStream) lack institutional-grade DCF and portfolio analytics, while ARGUS Enterprise and Cherre Agent.STUDIO operate on enterprise contracts inaccessible to small syndicators and family offices.
- **Excel still dominates** among small private equity funds and syndicators, despite organisations reporting 30–50% reductions in manual processing time after adopting investment management software.
- **Assumptions are still manual.** No incumbent automatically validates rent, vacancy, or cap rate assumptions against current submarket data; analysts research benchmarks by hand or rely on static figures.
- **The acquisition-to-operations loop is broken.** No single platform spans deal screening, underwriting, and ongoing portfolio tracking — investors stitch together multiple tools.
- **Incumbents are proprietary, expensive, and slow to evolve.** ARGUS has a steep learning curve, legacy UX, and no native AI assumption generation; Cherre Agent.STUDIO is enterprise-only with limited production track record.

---

## Key Features

### Deal Underwriting

- DCF cash flow modelling with configurable hold period, discount rate, exit cap rate, and financing terms
- Automated calculation of IRR, NPV, cash-on-cash, equity multiple, DSCR, and cap rate
- Support for multifamily, single-family buy-and-hold, fix-and-flip, and commercial (NNN, mixed-use) deal types
- Rent roll and T12 import (CSV/Excel) with automated model population
- Scenario modelling (bear/base/bull) with sensitivity tables

### AI-Augmented Analysis

- AI-powered assumption pre-fill using submarket data from integrated data providers (ATTOM, BatchData)
- Natural language deal entry — describe a deal in plain language and have the model auto-populate with sourced assumptions
- Agentic deal screener that scores inbound listings against fund-defined criteria
- Automated quarterly LP report generation with narrative commentary from an LLM

### Syndication & LP Reporting

- LP waterfall calculator for GP/LP syndication structures with investor-level returns
- Shareable investor-ready PDF and interactive report output
- Reg D compliance module (Form D tracking, investor accreditation status)
- NCREIF/PREA-compliant performance reporting for institutional fund managers

### Portfolio & Market Intelligence

- Portfolio dashboard aggregating performance across multiple assets
- Comparable transaction analysis with market benchmarking for assumption validation
- Integration pathway with property management systems (Yardi, MRI) and ARGUS API for institutional data exchange

---

## AI-Native Advantage

AI capabilities reposition this project against incumbents that still rely on static analyst inputs and manual research. Submarket-calibrated assumption generation pulls live transaction, rent, and vacancy data into underwriting models without analyst time. Natural-language deal entry lets analysts describe a deal in notes or speech and have a full model populated with sourced, traceable assumptions. Continuous acquisition-opportunity scoring ranks listed and off-market properties against a fund's investment criteria, while LP-reporting automation converts raw portfolio data into quarterly reports with narrative performance commentary.

---

## Tech Stack & Deployment

The project is intended as an open-source SaaS with self-hostable deployment for funds with data-residency or compliance constraints. An API-first architecture supports integration with property management systems (Yardi Voyager, MRI, RealPage) and the ARGUS API for data exchange with institutional counterparts. Standard return calculations (IRR, NPV, DCF) are mathematical and freely implementable; integrations with ATTOM, BatchData, and similar data providers supply submarket comps and assumption sourcing.

---

## Market Context

The US PropTech market is valued at approximately $25B in 2025, projected to reach $88B by 2032 at 11.9% CAGR; global proptech funding hit $16.7B in 2025, the highest level since 2019. Pricing in the segment is highly bifurcated: prosumer tools run from free to $100/month while institutional platforms (ARGUS, Cherre, CoStar Analytics at $200–$500/user/month) operate on enterprise contracts costing tens of thousands per year. Primary buyers include institutional asset managers and REITs, private equity real estate funds, syndicators, commercial brokers, and family offices building diversified portfolios without institutional-grade tooling.

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
