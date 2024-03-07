# Accessibility

*Accessibility* is my personal, programmatic accessibility scan dashboard. *Accessibility* relies on the excellent [Evaluatory project by Derek Kay](https://darekkay.com/evaluatory/) and GitHub Actions. Evaluatory packages the latest [axe-core](https://github.com/dequelabs/axe-core) and [html-validate](https://html-validate.org/) to provide clean, HTML-based scan reports. Screenshots of each page (at various widths) are provided as well.

:bar_chart: <https://code.jasonmorris.com/accessibility/> always shows the latest, timestamped report.

:warning: A URL with 0 issues does not validate accessibility thoroughly. These programmatic tests cover less than 50% of [WCAG Success Criteria](https://www.w3.org/WAI/WCAG21/quickref/). Manual testing is required to assess 100% of the guidelines.

## Installation

To test locally or contribute to the codebase:

1. `git clone https://github.com/jsnmrs/accessibility.git`
2. `cd accessibility`
3. `npm install`

## Running a local scan

`npm run scan`

## Adding or removing URLs

Edit [`config.json`](https://github.com/jsnmrs/accessibility/blob/main/config.json) to add or remove URLs from the scan list.

## Adjusting schedule

In [`.github/workflows/scan.yml`](https://github.com/jsnmrs/accessibility/blob/main/.github/workflows/scan.yml), edit the `schedule` using `crontab` syntax ([crontab.guru](https://crontab.guru/) is a helpful reference)

## Run an immediate scan

1. Visit the [`Run accessibility scan` GitHub Action history page](https://github.com/jsnmrs/accessibility/actions/workflows/scan.yml)
2. Click the "Run workflow" button
3. Select Use workflow from "Branch: main"
4. Click "Run workflow"
5. Look for the confirmation banner at the top of the page
6. Go to <https://code.jasonmorris.com/accessibility>
7. Results should update within 5 minutes
