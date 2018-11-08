# Before migrating Enjoy

* Update yourself with the latest OCA Conventions: https://github.com/OCA/maintainer-tools/blob/master/CONTRIBUTING.md
* Subscribe to the mailing list of the related project: https://odoo-community.org/groups
* Announce on the corresponding GitHub issue with the name "Migration to version 11.0" which module(s) you want to migrate

# Tasks to do in the migration

* Bump module version to `11.0.1.0.0`
* The first line `# -*- coding: utf-8 -*-` [is not needed anymore in Python 3](https://stackoverflow.com/questions/37048761/coding-utf-8-on-python3/37048998#37048998) if you save it in that encoding. (You can use the following command to auto-delete it: `find . -name "*.py" -exec sed -i "/#.*coding\: /d" {} \;`)
* Update README.rst from https://raw.githubusercontent.com/OCA/maintainer-tools/master/template/module/README.rst if not updated to the latest template.
* Convert Python 3 incompatible code. You can automate some things using 2to3 utility (bundled in most Linux distributions) with this command being on the module directory (it can require a later manual review for optimizing some sentences):
  ```bash
  2to3 -wnj4 --no-diffs .
  ```
  You can also check Python 3 compatibility and conversions guide by Odoo: https://github.com/odoo/odoo/blob/11.0/doc/python3.rst
* Remove any possible migration script from previous version.
* Remove the use of workflows (they have disappeared in this version).
* All area configs have been merged on a general `res.config.settings` model, so you have to adapt your possible settings in your module.
* For v10 `ir.values` entries:
  * if they are for showing an option under "Print" or "Actions" dropdown menu, remove the record and just add a field `binding_model_id` on the ir.actions.act_window linked record. Add `binding_type` = `report` if the ir.values had `key2` = `client_print_multi`.
  * if they are for having default values, use instead the model `ir.default`.
* If you use reports, `report` module has been split between `base` and `web`:
  * Change all references of `ir.actions.report.xml` by `ir.actions.report`, as the model has been changed. 
  * Replace t-call occurrences of base report templates:
    * `report.external_layout` > `web.external_layout`.
    * `report.external_layout_header`: No direct equivalent. You need to insert the changes inside `div class="header o_clean_header">` element of the `web.external_layout_?` view, being `?` one of the available "themes" (`background`, `boxed`, `clean` and `standard` in core)
    * `report.external_layout_footer`: the same as above, but looking inside `<div class="footer o_background_footer">` element.
    * `report.html_container` > `web.html_container`.
    * `report.layout` > `web.report_layout`.
    * `report.minimal_layout` > `web.minimal_layout`.
* Remove `group_ids` field in `ir.config_parameter` records, and read/write them with `sudo()` instead (https://github.com/odoo/odoo/commit/d0ca2d115e8b40308e1ff83bf9e7cc874762fbff).
* If using widget `kanban_state_selection` in any view, change it to just `state_selection`.
* If you create `ir.cron` in module setup you have to change:
    * `model` (string) -> `model_id` (m2o to model)
    * `function` (string) + `args` (string) -> `code` (eg: model.cron_that_does_something())
* Add tests to increase code coverage.
* Do the rest of the changes you need to do for making the module works on new version.


# Howto

## Technical method to migrate a module from "10.0" to "11.0" branch

* `$REPO`: the OCA repository hosting the module
* `$MODULE`: the name of the module you want to migrate
* `$USER_ORG`: your GitHub login or organization name

**Full process for beginners in Git flows**

* On a shell command:
  ```bash
  $ git clone https://github.com/OCA/$REPO -b 11.0
  $ git checkout -b 11.0-mig-$MODULE origin/11.0
  $ git format-patch --keep-subject --stdout origin/11.0..origin/10.0 -- $MODULE | git am -3 --keep
  ```
* Check https://github.com/OCA/maintainer-tools/wiki/Merge-commits-in-pull-requests for a procedure for reducing commits from "OCA Transbot...".
* Adapt the module to the 11.0 version following tasks to do.
* On a shell command:
  ```bash
  $ git add --all
  $ git commit -m "[MIG] $MODULE: Migration to 11.0"
  $ git remote add $USER_ORG git@github.com:$USER_ORG/$REPO.git # This mode requires an SSH key in the GitHub account
  $ ... or ....
  $ git remote add $USER_ORG https://github.com/$USER_ORG/$REPO.git # This will required to enter user/password each time
  $ git push $USER_ORG 11.0-mig-$MODULE --set-upstream
  ```

**Short method for advanced users that know the rest of the Git flow**

```bash
$ git remote update # In case the repo was already cloned before
$ git checkout -b 11.0-mig-<module> origin/11.0
$ git format-patch --keep-subject --stdout origin/11.0..origin/10.0 -- <module path> | git am -3 --keep
```

**Troubleshooting**

Sometimes, when performing these operations, the process can hang due to conflicts on the patches being applied. One of the possible problems is because a patch removes entirely a file and `git am` is not able to resolve this. It ends with a message `error: ...: patch does not apply`.

If that's your case, you can add `--ignore-whitespace` at the end of the `git am` command for allowing the patch to be applied, although there will be still conflicts to be solved, but they will be doable through standard conflict resolution system. Starting from the previous situation, you will need to do:

```bash
git am --abort
git format-patch --keep-subject --stdout origin/11.0..origin/10.0 -- <module path> | git am -3 --keep --ignore-whitespace
# resolve conflicts (for example git rm <file>)
git add --all
git am --continue
```

# Initialization (already done in OCA)

Before migrating the first module, the following tasks must be performed in the repository:

* Create empty 11.0 branch.
* Create metafiles (.travis.yml and README.md) from 10.0, adapting them to have the proper data.
* Make 11.0 default branch.

There's an script to automate all these process in: https://github.com/OCA/maintainer-tools/blob/master/tools/migrate_branch_empty.py.

Some repositories PSCs have decided to keep previous procedure putting installable flag on module as False, using this script: https://github.com/OCA/maintainer-tools/blob/master/tools/migrate_branch.py.
