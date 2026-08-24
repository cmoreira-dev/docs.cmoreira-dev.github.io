# docs.cmoreira-dev
HomeLab documentation

## Setup python environment

#### Create the virtual environment 'venv' using Python 3
````bash
python3 -m venv venv
````

#### Activate the virtual env for this project
````bash
source venv/bin/activate
````

#### Install dependencies
````bash
pip install -r requirements.txt
````

## Commands

Run these from inside `docs/` (where `mkdocs.yml` lives), with the venv active:

* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build --strict` - Build the documentation site, failing on broken links/nav entries.
* `mkdocs -h` - Print help message and exit.

## Project layout

    requirements.txt  # Python deps (mkdocs, mkdocs-material)
    docs/
        mkdocs.yml     # The configuration file.
        docs/
            index.md   # The documentation homepage.
            ...        # Other markdown pages, images and other files.
