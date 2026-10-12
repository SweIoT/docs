# sweiot-docs

Source for the SweIoT documentation site. Pages are Markdown files under `docs/`, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and published by Read the Docs on every push to `main`.

## Write locally

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -m mkdocs serve
```

Open http://127.0.0.1:8000. The page reloads as you save. `mkdocs build --strict` runs the same check as the deploy workflow.

## Add a page

1. Create a Markdown file under `docs/`, for example `docs/gateway/user-guide.md`.
2. Add it to the `nav` list in `mkdocs.yml`.
3. Put screenshots next to the page (for example `docs/gateway/img/`) and reference them with `![Alt text](img/name.png)`.

Useful syntax: `!!! note "Title"` for a callout, `=== "Admin UI"` / `=== "Mobile app"` for tabs. See the setup guide for examples.

## Publish for the first time

The site is hosted on [Read the Docs](https://readthedocs.org), which builds it from `.readthedocs.yaml` on every push.

1. Push the branch to `SweIoT/docs` on GitHub and merge it into `main` once the test build looks right. (Before this, `main` held the untouched Read the Docs tutorial template.)
2. Sign in to readthedocs.org with GitHub, choose **Add project**, and pick `SweIoT/docs`. Read the Docs detects `.readthedocs.yaml` and runs the first build. The site appears at `https://<project-slug>.readthedocs.io/`.
3. To test a branch before merging, open the project's **Versions** page, activate the branch and tick **Hidden**: it builds at `https://<project-slug>.readthedocs.io/en/<branch>/` without appearing in the public version menu. Pull requests get a preview build once **Build pull requests** is switched on under Settings.
4. Custom domain: in the project's **Domains** page add the host (for example `docs.sweiot.se`), then create a DNS `CNAME` record for it pointing at `readthedocs.io`. HTTPS is issued automatically. Finally set `site_url` in `mkdocs.yml` to the final address.

Read the Docs builds are private to logged-in team members only on the paid plan; on the free plan the site is public, which is fine for these guides.
