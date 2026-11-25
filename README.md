#!/usr/bin/env python3
"""Met à jour la section « Mes Projets » du README à partir de l'API GitHub.

Remplace le contenu entre <!-- PROJECTS:START --> et <!-- PROJECTS:END -->.
Variables d'environnement (toutes optionnelles) :
  GITHUB_USER   utilisateur à lister (défaut : Coco921609)
  GITHUB_TOKEN  jeton pour éviter les limites de l'API (fourni par Actions)
  MAX_PROJECTS  nombre de projets affichés (défaut : 6)
  EXCLUDE       noms de dépôts à ignorer, séparés par des virgules
"""
import json
import os
import re
import sys
import urllib.request

USER = os.environ.get("GITHUB_USER", "Coco921609")
TOKEN = os.environ.get("GITHUB_TOKEN")
MAX_PROJECTS = int(os.environ.get("MAX_PROJECTS", "6"))
EXCLUDE = {n.strip().lower() for n in os.environ.get("EXCLUDE", "").split(",") if n.strip()}
README = os.environ.get("README_PATH", "README.md")

START, END = "<!-- PROJECTS:START -->", "<!-- PROJECTS:END -->"


def fetch_repos():
    url = f"https://api.github.com/users/{USER}/repos?per_page=100&type=owner&sort=pushed"
    req = urllib.request.Request(url, headers={
        "Accept": "application/vnd.github+json",
        "User-Agent": "readme-updater",
        **({"Authorization": f"Bearer {TOKEN}"} if TOKEN else {}),
    })
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.load(resp)


def select(repos):
    keep = [
        r for r in repos
        if not r.get("fork")
        and not r.get("archived")
        and r["name"].lower() != USER.lower()      # le dépôt du profil lui-même
        and r["name"].lower() not in EXCLUDE
    ]
    keep.sort(key=lambda r: r["pushed_at"], reverse=True)
    return keep[:MAX_PROJECTS]


def cell(text, limit=110):
    text = (text or "—").replace("\n", " ").replace("|", "\\|").strip()
    return text if len(text) <= limit else text[: limit - 1].rstrip() + "…"


def render(repos):
    if not repos:
        return "_Aucun projet public pour le moment._"
    lines = [
        "| Projet | Description | Langage | ⭐ | Dernière activité |",
        "| :--- | :--- | :---: | :---: | :---: |",
    ]
    for r in repos:
        lines.append(
            f"| [**{r['name']}**]({r['html_url']}) "
            f"| {cell(r.get('description'))} "
            f"| {r.get('language') or '—'} "
            f"| {r.get('stargazers_count', 0)} "
            f"| {r['pushed_at'][:10]} |"
        )
    return "\n".join(lines)


def update(text, block):
    pattern = re.compile(re.escape(START) + r".*?" + re.escape(END), re.DOTALL)
    if not pattern.search(text):
        sys.exit(f"Marqueurs {START} / {END} introuvables dans {README}")
    return pattern.sub(lambda _: f"{START}\n{block}\n{END}", text)


def main():
    block = render(select(fetch_repos()))
    with open(README, encoding="utf-8") as f:
        old = f.read()
    new = update(old, block)
    if new == old:
        print("README déjà à jour.")
        return
    with open(README, "w", encoding="utf-8") as f:
        f.write(new)
    print("README mis à jour.")


if __name__ == "__main__":
    main()

[![Consulter mes projets sur GitHub](https://img.shields.io/badge/Consulter_mes_projets_sur_GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Coco921609?tab=repositories)

---

*⚡ Le code d'aujourd'hui construit les grands projets de demain.*
