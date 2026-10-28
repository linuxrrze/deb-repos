# deb-repos

Unofficial APT repositories for software that ships `.deb` files but has no APT repository of its own:

| Repository   | Packages                                                     | Architectures          | .deb files served from |
|--------------|--------------------------------------------------------------|------------------------|------------------------|
| `zoom`       | the [Zoom](https://zoom.us) desktop client (`zoom`)          | amd64                  | `cdn.zoom.us`          |
| `storytold`  | the [storytold](https://github.com/storytold) "Craft" apps (`photocraft`, `filmcraft`, `wordcraft`, …) | amd64, arm64, riscv64 | their GitHub releases |
| `solidcraft` | [SolidCraft](https://github.com/tangency-labs/solidcraft), parametric 3D CAD (`solidcraft`) | amd64, arm64 | its GitHub releases |

GitHub Actions builds the signed indexes and publishes them on GitHub Pages, one directory per repository.
The `.deb` files are **not** copied: apt downloads them directly from where the vendor publishes them.
All three repositories are signed with the same key.

# Installation

Requires apt 1.6 or newer (Ubuntu 18.04 / Debian 10 or later). First get the key:

```sh
sudo install -d -m 0755 /etc/apt/keyrings
sudo curl -fsSLo /etc/apt/keyrings/deb-repos.asc https://linuxrrze.github.io/deb-repos/deb-repos.asc
```

Check the key before trusting it: `gpg --show-keys /etc/apt/keyrings/deb-repos.asc`.

Then add the repositories you want. For each, replace `<repo>` with `zoom`, `storytold` or `solidcraft`:

```sh
sudo tee /etc/apt/sources.list.d/<repo>.sources <<EOF
Types: deb
URIs: mirror+https://linuxrrze.github.io/deb-repos/<repo>/mirrors.txt
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/deb-repos.asc
EOF
```

and install:

```sh
sudo apt update
sudo apt install zoom                                # zoom
sudo apt install photocraft filmcraft wordcraft      # storytold; `apt search craft` lists them all
sudo apt install solidcraft                          # solidcraft
```

A package that was installed by hand from a downloaded `.deb` has the same name, so `apt upgrade` (or
unattended-upgrades) now keeps it up to date. The Origins for unattended-upgrades are `zoom-ubuntu-repo`,
`storytold-mirror` and `solidcraft-mirror`.

## Older versions

Every repository keeps older versions, so you can downgrade:

```sh
apt list -a solidcraft                   # show all available versions
sudo apt install solidcraft=0.25.0-1     # install a specific version
sudo apt-mark hold solidcraft            # keep it from being upgraded (undo with apt-mark unhold)
```

- `zoom`: the 10 newest Zoom versions when the repository started, plus every release since.
- `storytold`: the last 5 versions of each app.
- `solidcraft`: the last 10 versions.

## Removal

```sh
sudo apt remove <packages>
sudo rm /etc/apt/sources.list.d/<repo>.sources
sudo rm /etc/apt/keyrings/deb-repos.asc      # once no repository uses it anymore
sudo apt update
```

# How it works

```
github.com/linuxrrze/deb-repos      GitHub Actions, every 3 hours        GitHub Pages: linuxrrze.github.io/deb-repos/
  zoom/  stanzas, versions.txt  ──▶  zoom/scripts/add-new-versions.sh ──▶   deb-repos.asc
  storytold/state/debs.json          github-apt-mirror.py (storytold)       zoom/        mirrors.txt, dists/stable/...
  solidcraft/state/debs.json         github-apt-mirror.py (solidcraft)      storytold/   mirrors.txt, dists/stable/...
                               ◀──   commit new index entries              solidcraft/  mirrors.txt, dists/stable/...
```

- Each `mirrors.txt` is a list for apt's `mirror+` method: index files are fetched from GitHub Pages
  (`type:index`), `.deb` files from `https://cdn.zoom.us/` or `https://github.com/` (`type:deb`). Each
  `Filename:` in an index is the path of the `.deb` there, e.g.
  `tangency-labs/solidcraft/releases/download/v0.25.1/solidcraft-0.25.1-linux-x86_64.deb`.
- The indexes are signed and list the SHA-256 of every package, so apt rejects a `.deb` that differs from the
  one that was checked when it was added.
- Before a version is added, its `.deb` is downloaded once and checked: Zoom packages against Zoom's own
  signature (see [zoom/README.md](zoom/README.md)), packages from GitHub releases against the SHA-256 the
  release publishes (GitHub's asset digest, or `SHA256SUMS.txt`). The `.deb` itself isn't stored, only its index
  entry, which is committed to this repository.
- `.github/workflows/update.yml` runs every 3 hours (and on pushes). It adds new versions to the three
  repositories one after the other, commits their index entries, then signs and deploys all indexes.
  If one repository or release fails, everything else is still published and the run is marked as failed;
  the next run retries. Nothing is deployed if one of the repositories has no index at all.
- After 50 days without changes, the workflow pushes an empty keepalive commit, because GitHub disables
  scheduled workflows after 60 days without repository activity.

## GitHub releases: `github-apt-mirror.py`

`storytold` and `solidcraft` are built by `github-apt-mirror.py`. For the given org (or some of its repos), it

1. looks at the **latest stable release** (or the newest `--releases N`) of every non-archived repo, pushed to
   within the last year, that has `.deb` assets (prereleases and drafts are ignored),
2. downloads the new `.deb` files and checks each against its published SHA-256 (a `.deb` that is attached a
   second time under another name, e.g. `solidcraft-linux-x86_64.deb`, is only indexed once),
3. keeps the newest `--keep` versions per package and architecture,
4. writes `Packages`/`Release` indexes, signs them with your GPG key and swaps them into place.

The release file names map `x86_64` → `amd64`, `aarch64` → `arm64` and `riscv64` → `riscv64`.

`storytold` is indexed with `--org storytold`, which finds all of its apps, `solidcraft` with
`--org tangency-labs --repos solidcraft --releases 10`, because SolidCraft publishes several releases a day and
only looking at the latest one every 3 hours could miss some.

To add another project that publishes `.deb` files with its GitHub releases, add a step like the `solidcraft`
one to the workflow, its `<name>/state/debs.json` to the `git add` line and the path triggers, and the name to
the completeness check.

Dependencies: `python3` (stdlib only), `dpkg` (for `dpkg-deb`), `gpg`. Nothing from PyPI.

```
--out DIR            repository root that the web server serves (required)
--org ORG            GitHub organization or user (required)
--repos A B ...      only these repos (default: discover every repo with .deb releases)
--releases N         look at the newest N stable releases of each repo (default 1, the latest);
                     must not be larger than --keep
--max-age-days N     discovery skips repos not pushed to in N days (default 365)
--arch amd64 ...     only these architectures (amd64, arm64, riscv64); packages for
                     other architectures are removed from the pool
--keep N             versions kept per package/arch (default 2)
--gpg-key FPR        signing key; without one the repo is unsigned
--name NAME          keyring file name <NAME>-archive-keyring.{asc,gpg} (default: --org)
--origin NAME        Origin/Label of the Release file (default: <NAME>-mirror)
--suite NAME         default "stable"
--no-fetch           only rebuild the indexes from what's already indexed
--allow-unverified   accept assets that have no published checksum
--remote-debs URL    don't store the .debs, send clients to the GitHub releases for them
                     (writes mirrors.txt); URL is where --out is served
--state DIR          package metadata; with --remote-debs the record of all indexed
                     versions (default: OUT/.state)
```

The script exits with status 1 if a repo or asset couldn't be fetched; everything else is still published.
Set `GITHUB_TOKEN` to raise the GitHub API limit from 60 to 5000 requests per hour (the workflow does).

## Zoom

Zoom has no GitHub releases and only reveals its latest version, so the `zoom` repository is built by its own
scripts, see [zoom/README.md](zoom/README.md).

# Setting up your own copy (fork)

1. Fork the repository and enable GitHub Pages: Settings → Pages → Source: **GitHub Actions**.
2. Create a signing key without passphrase, in a temporary GnuPG home outside your checkout:
   ```sh
   export GNUPGHOME=$(mktemp -d)
   gpg --batch --passphrase '' --quick-gen-key "deb-repos APT repositories <you@example.org>" ed25519 sign 5y
   gpg --armor --export-secret-keys > "$GNUPGHOME/private.asc"
   ```
3. Store the content of `$GNUPGHOME/private.asc` as Actions secret `REPO_GPG_PRIVATE_KEY`
   (Settings → Secrets and variables → Actions). Keep an offline backup, then `rm -rf "$GNUPGHOME"`.
4. Run the "Update repositories" workflow (Actions tab). The first run indexes the newest releases, which
   downloads each `.deb` once (several GB). It publishes to `https://<user>.github.io/<repository>/`.
5. Replace `linuxrrze.github.io/deb-repos` in this README with your own URL, and add your key's fingerprint
   to the installation instructions.

Before the key expires, extend it (`gpg --quick-set-expire`), update the secret and let users download the key
again.

# Self-hosted repositories

## GitHub releases (Debian/Ubuntu, nginx, systemd)

To store the `.deb` files on your own server, run `github-apt-mirror.py` without `--remote-debs`. The systemd
units in `deploy/` are templates, one instance per repository; the instance name is the directory below
`/srv/apt`:

```sh
sudo apt install python3 dpkg gnupg nginx
sudo useradd --system --home /var/lib/aptmirror --create-home aptmirror
sudo install -m755 github-apt-mirror.py /usr/local/bin/

# Signing key (no passphrase, because the timer signs unattended)
sudo -u aptmirror env GNUPGHOME=/var/lib/aptmirror/gnupg \
  gpg --batch --passphrase '' --quick-gen-key "APT mirror <apt@example.org>" ed25519 sign never
FPR=$(sudo -u aptmirror env GNUPGHOME=/var/lib/aptmirror/gnupg \
  gpg --list-keys --with-colons | awk -F: '/^fpr/{print $10; exit}')

# One settings file per repository
sudo install -d -o aptmirror -g aptmirror /srv/apt/storytold /srv/apt/solidcraft
sudo tee /etc/default/github-apt-mirror-storytold <<EOF
SIGNING_KEY=$FPR
MIRROR_ARGS=--org storytold --arch amd64 arm64 riscv64 --keep 2
#GITHUB_TOKEN=github_pat_...
EOF
sudo tee /etc/default/github-apt-mirror-solidcraft <<EOF
SIGNING_KEY=$FPR
MIRROR_ARGS=--org tangency-labs --repos solidcraft --name solidcraft --arch amd64 arm64 --releases 3 --keep 3
#GITHUB_TOKEN=github_pat_...
EOF
sudo chmod 600 /etc/default/github-apt-mirror-*

sudo cp deploy/github-apt-mirror@.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl start github-apt-mirror@storytold.service    # first sync, can take a while
journalctl -u github-apt-mirror@storytold -e
sudo systemctl enable --now github-apt-mirror@storytold.timer github-apt-mirror@solidcraft.timer  # hourly

sudo cp deploy/nginx-apt.conf /etc/nginx/sites-available/apt   # edit server_name / TLS
sudo ln -s ../sites-available/apt /etc/nginx/sites-enabled/ && sudo systemctl reload nginx
```

Without a `GITHUB_TOKEN`, GitHub allows 60 API requests per hour. A storytold run uses one request for the
repo list plus one per recently active repo (about 30 today), so an hourly run fits for now. If you sync more
repositories or more often, create a fine-grained token with no permissions and add it as shown above.

Disk space: one version of every storytold app for all three architectures takes several GB. Use fewer
architectures and/or `--keep 1` to use less.

Clients:

```sh
sudo curl -fsSLo /usr/share/keyrings/storytold-archive-keyring.gpg \
  https://apt.example.org/storytold/storytold-archive-keyring.gpg

sudo tee /etc/apt/sources.list.d/storytold.sources <<EOF
Types: deb
URIs: https://apt.example.org/storytold
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /usr/share/keyrings/storytold-archive-keyring.gpg
EOF
```

(and the same with `solidcraft` instead of `storytold`).

## Zoom

See [zoom/README.md](zoom/README.md#self-hosted-repository).

# Caveats

- These are unofficial repositories. You're trusting the vendors' downloads (checksums only show the file
  arrived intact; Zoom's packages are additionally signed by Zoom), and you trust the signing key of this
  repository.
- The storytold and SolidCraft packages' `Depends` fields target recent distros (e.g. `libc6 >= 2.35`, i.e.
  Ubuntu 22.04 / Debian 12 or newer). On older systems apt will refuse to install them.
- A newer version that is still marked as a prerelease on GitHub (e.g. SolidCraft's nightlies) isn't picked up.
- If a release or asset is deleted upstream, its version stays in the index until it's pruned, but can't be
  installed anymore.
