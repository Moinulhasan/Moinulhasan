#!/usr/bin/env python3
"""
Builds assets/stats.svg from the GitHub GraphQL API.

Runs inside GitHub Actions with the default GITHUB_TOKEN. No third-party
service, so there is nothing to rate-limit us and nothing to go down.
"""

import json
import os
import sys
import urllib.request
from datetime import datetime, timezone

LOGIN = os.environ.get("PROFILE_LOGIN", "Moinulhasan")
TOKEN = os.environ["GITHUB_TOKEN"]
OUT = os.environ.get("STATS_OUT", "assets/stats.svg")

BASE, PANEL, RULE = "#0E2228", "#12262B", "#1D3B42"
TEXT, MUTED, FAINT = "#EDF2F0", "#7A9BA1", "#5F868D"
RED, AMBER, GREEN = "#FF3B2F", "#F0B429", "#35C08A"
MONO = "ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"

QUERY = """
query($login: String!) {
  user(login: $login) {
    followers { totalCount }
    contributionsCollection {
      totalCommitContributions
      totalPullRequestContributions
      totalIssueContributions
      totalPullRequestReviewContributions
      contributionCalendar { totalContributions }
    }
    repositories(first: 100, ownerAffiliations: OWNER, isFork: false,
                 orderBy: {field: STARGAZERS, direction: DESC}) {
      totalCount
      nodes {
        stargazerCount
        languages(first: 10, orderBy: {field: SIZE, direction: DESC}) {
          edges { size node { name color } }
        }
      }
    }
  }
}
"""


def fetch():
    body = json.dumps({"query": QUERY, "variables": {"login": LOGIN}}).encode()
    req = urllib.request.Request(
        "https://api.github.com/graphql",
        data=body,
        headers={
            "Authorization": f"bearer {TOKEN}",
            "Content-Type": "application/json",
            "User-Agent": f"{LOGIN}-profile-stats",
        },
    )
    with urllib.request.urlopen(req, timeout=30) as r:
        payload = json.load(r)
    if "errors" in payload:
        sys.exit(f"GitHub API error: {payload['errors']}")
    return payload["data"]["user"]


def esc(s):
    return (s.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;"))


def human(n):
    return f"{n:,}"


def build(user):
    cc = user["contributionsCollection"]
    repos = user["repositories"]["nodes"]

    stars = sum(r["stargazerCount"] for r in repos)
    totals = {}
    for r in repos:
        for e in r["languages"]["edges"]:
            name = e["node"]["name"]
            if name not in totals:
                totals[name] = {"size": 0, "color": e["node"]["color"] or "#7A9BA1"}
            totals[name]["size"] += e["size"]

    ranked = sorted(totals.items(), key=lambda kv: kv[1]["size"], reverse=True)[:6]
    grand = sum(v["size"] for _, v in ranked) or 1

    figures = [
        (human(cc["contributionCalendar"]["totalContributions"]), "contributions", "past year"),
        (human(cc["totalCommitContributions"]), "commits", "authored"),
        (human(cc["totalPullRequestContributions"]), "pull requests", "opened"),
        (human(cc["totalPullRequestReviewContributions"]), "reviews", "given"),
        (human(user["repositories"]["totalCount"]), "repositories", "not forks"),
        (human(stars), "stars", "earned"),
    ]

    W, H = 880, 320
    p = [
        f'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {W} {H}" width="{W}" '
        f'height="{H}" role="img" aria-label="GitHub activity for {esc(LOGIN)}">',
        f'<defs><linearGradient id="bgs" x1="0" y1="0" x2="0" y2="1">'
        f'<stop offset="0" stop-color="{BASE}"/><stop offset="1" stop-color="#0A181C"/>'
        f'</linearGradient></defs>',
        f'<rect width="{W}" height="{H}" rx="14" fill="url(#bgs)" stroke="{RULE}"/>',
        f'<text x="40" y="38" font-family="{MONO}" font-size="15" fill="{TEXT}">'
        f'what the last twelve months looked like</text>',
    ]

    stamp = datetime.now(timezone.utc).strftime("%d %b %Y")
    p.append(
        f'<text x="840" y="38" text-anchor="end" font-family="{MONO}" font-size="11" '
        f'fill="{FAINT}">rebuilt {stamp}</text>'
    )
    p.append(f'<line x1="40" y1="50" x2="840" y2="50" stroke="{RULE}"/>')

    # figures, two rows of three
    colw = (800 - 2 * 16) / 3
    for i, (value, label, sub) in enumerate(figures):
        col, row = i % 3, i // 3
        x = 40 + col * (colw + 16)
        y = 74 + row * 84
        accent = [RED, AMBER, GREEN][col]
        p.append(f'<rect x="{x:.0f}" y="{y}" width="3" height="56" fill="{accent}"/>')
        p.append(
            f'<text x="{x + 18:.0f}" y="{y + 30}" font-family="{MONO}" font-size="28" '
            f'fill="{TEXT}">{value}</text>'
        )
        p.append(
            f'<text x="{x + 18:.0f}" y="{y + 48}" font-family="{MONO}" font-size="12" '
            f'fill="{MUTED}">{label} <tspan fill="{FAINT}">{sub}</tspan></text>'
        )

    # language bar
    by = 258
    p.append(
        f'<text x="40" y="{by - 12}" font-family="{MONO}" font-size="11" '
        f'fill="{FAINT}">what the code is written in</text>'
    )
    x = 40.0
    bar_w = 800.0
    for i, (name, meta) in enumerate(ranked):
        w = bar_w * meta["size"] / grand
        r0 = 5 if i == 0 else 0
        r1 = 5 if i == len(ranked) - 1 else 0
        p.append(
            f'<path d="M{x + r0:.1f} {by} h{max(w - r0 - r1, 0.1):.1f} '
            f'a{r1} {r1} 0 0 1 {r1} {r1} v{14 - 2 * r1} a{r1} {r1} 0 0 1 -{r1} {r1} '
            f'h-{max(w - r0 - r1, 0.1):.1f} a{r0} {r0} 0 0 1 -{r0} -{r0} v-{14 - 2 * r0} '
            f'a{r0} {r0} 0 0 1 {r0} -{r0} z" fill="{meta["color"]}" opacity="0.9"/>'
        )
        x += w

    lx = 40.0
    for name, meta in ranked:
        pct = 100.0 * meta["size"] / grand
        p.append(f'<circle cx="{lx + 5:.0f}" cy="{by + 40}" r="5" fill="{meta["color"]}"/>')
        p.append(
            f'<text x="{lx + 18:.0f}" y="{by + 45}" font-family="{MONO}" font-size="12" '
            f'fill="{MUTED}">{esc(name)} <tspan fill="{FAINT}">{pct:.0f}%</tspan></text>'
        )
        lx += 22 + 8.0 * (len(name) + 5)

    p.append("</svg>")
    return "\n".join(p)


if __name__ == "__main__":
    svg = build(fetch())
    os.makedirs(os.path.dirname(OUT) or ".", exist_ok=True)
    with open(OUT, "w", encoding="utf-8") as f:
        f.write(svg + "\n")
    print(f"wrote {OUT} ({len(svg)} bytes)")