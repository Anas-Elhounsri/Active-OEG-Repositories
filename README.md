# Repository Activity Report Explanation

This README explains what the repository activity report measures, how the
activity of each repository is gauged, and what the different classifications
and fields mean.

---

## 1. What this report answers

For a list of repositories (typically the `repositories.json`), the report answers one question:

> **Is this repository still active and being maintained?**

It does this by looking at three publicly-visible signals on GitHub and
turning them into a single per-repository verdict.

---

## 2. The three activity signals

For each repository we query the GitHub REST API for three things:

| Signal | What it is | GitHub source |
| --- | --- | --- |
| **Last push** | When the repository's default branch last received commits. | The repository's `pushed_at` field. |
| **Last pull request** | When the most recent PR was opened (and, if applicable, merged). | The most recently *created* PR (`/pulls`). |
| **Latest release** | The most recent tagged release and when it was published. | The latest release (`/releases/latest`). |

Each of these yields a **timestamp** (or "none" if the repository never had
that kind of event — for example, many repositories have no releases at all).

We also read two GitHub flags that are explicit "unmaintained" markers:

- **`archived`**: the owner archived the repository (read-only, no longer
  accepting changes).
- **`disabled`**: GitHub disabled the repository.

---

## 3. The two time windows

The report uses two configurable windows to decide how recent activity is:

- **Active window**: default **6 months**.
- **Stale window**: default **24 months**.

> ⚠️ These are **approximate**: "months" are counted as 30-day periods, so
> 6 months ≈ 180 days and 24 months ≈ 720 days.

### What "24-month stale" means

The **stale window (24 months)** is the boundary between *"there was *some*
activity, but a while ago"* and *"nothing has happened for a very long time"*.

Concretely:

- A repository whose **most recent** activity happened **within the last
  6 months** is considered **active**.
- A repository whose most recent activity happened **between 6 and 24 months
  ago** is considered **low activity** (it was touched in the last two years,
  but not recently).
- A repository whose most recent activity is **older than 24 months** (i.e.,
  no push, PR, or release in the last two years) is considered **inactive**.

So "24-month stale" is the cutoff for "this has gone stale enough that we call
it inactive". Anything between the 6-month and 24-month marks is the grey zone
("low activity").

---

## 4. Classification rules

Each repository ends up in exactly one of five buckets. The decision is made
in this order:

1. **`archived`**: the GitHub `archived` or `disabled` flag is set. This
   takes precedence over everything else, even if the repository was touched
   recently.
2. **`active`**: any of the three signals (last push, last PR, latest release)
   falls within the last **6 months**.
3. **`low_activity`**: nothing within 6 months, but the most recent signal
   falls within the last **24 months**.
4. **`inactive`**: no signal within 24 months (or the repository has no
   push/PR/release data at all).
5. **`error`**: the repository could not be queried (deleted, renamed, made
   private, or not a GitHub URL).

The table:

| Classification | Meaning |
| --- | --- |
| `active` | Activity (push, PR, or release) within the last 6 months. |
| `low_activity` | Last activity between 6 and 24 months ago. |
| `inactive` | No activity in the last 24 months. |
| `archived` | GitHub `archived`/`disabled` flag set. |
| `error` | Could not be queried (deleted / renamed / private / non-GitHub). |

---

## 5. Contributors

For every repository classified as **`active`**, the report also collects the
list of **contributors** from GitHub (login, account type, number of
contributions, and profile URLs).

These are then **aggregated** across all active repositories, so you can see,
for example, which people contribute to more than one active project.

Two caveats:

- Contributors are only collected for `active` repositories.
- GitHub returns at most the **top 100** contributors per repository.

---

## 6. Output files and their fields

The script writes five files:

| File | Contents |
| --- | --- |
| `repository_activity.json` | Full per-repository records + a summary by classification. |
| `repository_activity.csv` | Flat, spreadsheet-friendly rows. |
| `repository_activity.md` | Human-readable report (summary + per-bucket tables + contributors). |
| `contributors.json` | Contributors aggregated across active repositories. |
| `contributors.csv` | Flat contributor rows. |

### Key fields per repository

| Field | Meaning |
| --- | --- |
| `name` / `url` | The repository name and GitHub URL from the input file. |
| `archived` / `disabled` | GitHub's "unmaintained" flags. |
| `pushed_at` | Last push timestamp. |
| `last_pr` | Most recent PR (`number`, `title`, `created_at`, `merged_at`). |
| `latest_release` | Latest release (`tag_name`, `published_at`). |
| `activity` | Boolean flags for each signal within the active window, plus the overall `last_event_at`. |
| `classification` | One of `active` / `low_activity` / `inactive` / `archived` / `error`. |
| `contributors` | List of contributors (only populated for `active` repos). |
| `error` | Explanation when the classification is `error`. |

### Key fields per contributor (aggregated)

| Field | Meaning |
| --- | --- |
| `login` | GitHub username. |
| `type` | `User` or `Bot`/`Organization`. |
| `html_url` / `avatar_url` | Profile and avatar links. |
| `repository_count` | Number of active repositories this person contributes to. |
| `total_contributions` | Sum of contributions across those active repositories. |

---

## 8. Worked example

Suppose today is **23 September 2026**.

- **Active window (6 months)** ends around **23 March 2026**.
- **Stale window (24 months)** ends around **23 September 2024**.

A repository whose last push was **June 2026** → `active`.
A repository whose last PR was **January 2025** (17 months ago) → `low_activity`
(within 24 months, but older than 6 months).
A repository whose last release was **2021** and no pushes since → `inactive`.
A repository flagged `archived` on GitHub → `archived`, regardless of dates.
A repository URL that no longer exists → `error`.
