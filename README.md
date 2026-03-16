# How to Delete a Google Workspace / Admin Account When Google Says Google Cloud Resources Still Exist

This guide documents a real-world fix for a confusing Google Workspace deletion loop:

- You try to delete the Google Workspace / Admin account
- Google Admin says you must first delete Google Cloud resources
- Google Cloud may show no normal projects in the UI
- The only visible item may be the organization/domain itself
- You cannot manually delete the organization node
- The real blockers are hidden Google-managed resources such as:
  - hidden `sys-*` projects
  - hidden folders like `system-gsuite`
  - nested folders like `apps-script`
 
This problem is especially frustrating given that this issue has existed for a while and no fix has been issued. Support might or Gemini's chatbot might loop you through a predefined script, or you might not have support at all if you have cancelled your billing. This guide will help you with that.

In short: **the Google Cloud UI may look empty, while hidden resources still exist and still block Workspace deletion**. The most reliable way to find and remove them is with **Cloud Shell** and `gcloud`.

> **Important**
>
> You cannot manually delete the Google Cloud **organization** resource itself.  
> Once all blocking Cloud resources are gone and the Workspace account deletion succeeds, Google removes the organization automatically.

---

## Table of Contents

- [Overview](#overview)
- [What You Need Before Starting](#what-you-need-before-starting)
- [Required Permissions](#required-permissions)
- [Key Caveats](#key-caveats)
- [Step 1: Open Google Cloud Shell](#step-1-open-google-cloud-shell)
- [Step 2: Confirm the Active Account and Clear Bad `gcloud` Project Config](#step-2-confirm-the-active-account-and-clear-bad-gcloud-project-config)
- [Step 3: Find Your Organization ID](#step-3-find-your-organization-id)
- [Step 4: Check Your Current Organization Roles](#step-4-check-your-current-organization-roles)
- [Step 5: Assign All Required Roles Up Front](#step-5-assign-all-required-roles-up-front)
- [Step 6: List All Projects, Including Hidden `sys-*` Projects](#step-6-list-all-projects-including-hidden-sys--projects)
- [Step 7: Delete Hidden `sys-*` Projects](#step-7-delete-hidden-sys--projects)
- [Step 8: Verify Project Deletion](#step-8-verify-project-deletion)
- [Step 9: If Project Deletion Fails, Fix Permissions](#step-9-if-project-deletion-fails-fix-permissions)
- [Step 10: Find Hidden Folders That May Not Appear in the UI](#step-10-find-hidden-folders-that-may-not-appear-in-the-ui)
- [Step 11: Check Hidden Folders for Remaining Projects](#step-11-check-hidden-folders-for-remaining-projects)
- [Step 12: Delete Hidden Folders Bottom-Up](#step-12-delete-hidden-folders-bottom-up)
- [Step 13: If Folder Deletion Fails, Fix Folder Permissions](#step-13-if-folder-deletion-fails-fix-folder-permissions)
- [Step 14: Check for Liens](#step-14-check-for-liens)
- [Step 15: Deep Scan for Remaining Assets](#step-15-deep-scan-for-remaining-assets)
- [Step 16: Final Console Verification](#step-16-final-console-verification)
- [Step 17: Delete the Workspace / Admin Account](#step-17-delete-the-workspace--admin-account)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [Summary](#summary)

---

## Overview

The issue usually looks like this:

1. You go to Google Admin and try to delete the Workspace account or organization.
2. Google Admin says you still have Google Cloud resources.
3. Google Cloud Resource Manager appears empty, or only shows the organization/domain.
4. You cannot delete the organization manually.
5. The hidden blockers turn out to be internal Cloud resources created by Google services.

A very common hidden structure is:

```text
Organization
└── system-gsuite
    └── apps-script
        └── sys-* projects
```

The reliable cleanup order is:

1. Delete hidden `sys-*` projects
2. Delete nested folders such as `apps-script`
3. Delete parent folders such as `system-gsuite`
4. Retry deletion in Google Admin, which will succeed if you've never used Google Cloud or have no existing resources in there.

---

## What You Need Before Starting

You should have:

- access to the Google account that is the Workspace admin you want to delete
- access to Google Cloud Console for that account
- enough IAM permission to view and modify organization IAM bindings

This guide assumes you can open Cloud Shell and run `gcloud` commands.

---

## Required Permissions

Assign these roles on the **organization** before doing cleanup:

- `roles/resourcemanager.organizationAdmin`
- `roles/resourcemanager.folderViewer`
- `roles/resourcemanager.projectDeleter`
- `roles/resourcemanager.folderAdmin`

What each one is needed for:

- **Organization Admin**: organization-level management
- **Folder Viewer**: lets you see hidden folder structure
- **Project Deleter**: lets you delete hidden `sys-*` projects
- **Folder Admin**: lets you delete hidden folders like `system-gsuite` and `apps-script`

> **Note**
>
> Some accounts may already have broader roles like `roles/owner`, but the roles above are the ones directly relevant to this workflow.

---

## Key Caveats

### The UI may not show the real blockers

The Google Cloud UI may show:

- no projects
- no child resources
- only the organization/domain

That does **not** mean the resources are gone.

Use Cloud Shell and `gcloud` to verify the real state.

### Project deletion uses the project ID, not the project number

A deletion command must use something like:

```text
sys-67522705716024422684697340
```

not the numeric project number like:

```text
1076591021334
```

### Hidden folders may not appear in the UI tree

Even if the resource tree does not expand, hidden folders can still exist and still block deletion.

### `DELETE_REQUESTED` is usually acceptable

When you delete projects or folders, Google may mark them as `DELETE_REQUESTED` before they fully disappear. That is usually expected.

### Cloud Shell can be misconfigured

If `gcloud` is set to use a **project number** as the default project, some commands will fail until you clear that config.

---

## Step 1: Open Google Cloud Shell

1. Open Google Cloud Console:
   - `https://console.cloud.google.com/`
2. Sign in with the Workspace admin account you are trying to delete.
3. In the top-right area of the page, click the **Activate Cloud Shell** button.
   - It looks like a terminal icon: `>_`
4. Wait for the shell panel to open.

All steps below can be done from this shell.

---

## Step 2: Confirm the Active Account and Clear Bad `gcloud` Project Config

First, check which account is authenticated:

```bash
gcloud auth list
```

Get the current active account:

```bash
gcloud config get-value account
```

Save it into a variable:

```bash
export ADMIN_EMAIL="$(gcloud config get-value account)"
echo "$ADMIN_EMAIL"
```

Now clear any default project setting. This is important because some `gcloud` commands fail if `core/project` is set to a project number instead of a project ID:

```bash
gcloud config unset project
```

Verify your config:

```bash
gcloud config list
```

If you ever see an error like this:

```text
The value of `core/project` property is set to project number...
```

run:

```bash
gcloud config unset project
```

---

## Step 3: Find Your Organization ID

List organizations visible to the account:

```bash
gcloud organizations list
```

You should see your domain and a numeric organization ID.

Example output:

```text
DISPLAY_NAME: alt.interactiva.studio
ID: 604316355852
DIRECTORY_CUSTOMER_ID: Cxxxxxxx
```

Save the organization ID in a variable:

```bash
export ORG_ID="YOUR_ORG_ID_HERE"
```

Example:

```bash
export ORG_ID="604316355852"
```

Verify it:

```bash
echo "$ORG_ID"
```

---

## Step 4: Check Your Current Organization Roles

Run:

```bash
gcloud organizations get-iam-policy "$ORG_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$ADMIN_EMAIL" \
  --format="table(bindings.role)"
```

Look for the roles listed earlier:

- `roles/resourcemanager.organizationAdmin`
- `roles/resourcemanager.folderViewer`
- `roles/resourcemanager.projectDeleter`
- `roles/resourcemanager.folderAdmin`

If one or more are missing, assign them in the next step.

---

## Step 5: Assign All Required Roles Up Front

If your account already has enough permission to change org IAM, assign all needed roles now:

```bash
for ROLE in \
  roles/resourcemanager.organizationAdmin \
  roles/resourcemanager.folderViewer \
  roles/resourcemanager.projectDeleter \
  roles/resourcemanager.folderAdmin
do
  gcloud organizations add-iam-policy-binding "$ORG_ID" \
    --member="user:$ADMIN_EMAIL" \
    --role="$ROLE"
done
```

Verify the result:

```bash
gcloud organizations get-iam-policy "$ORG_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$ADMIN_EMAIL" \
  --format="table(bindings.role)"
```

> **Important**
>
> If these commands fail with permission errors, you need another org admin/owner to assign the roles to your account before continuing.

---

## Step 6: List All Projects, Including Hidden `sys-*` Projects

List all projects with both IDs and numbers:

```bash
gcloud projects list --format="table(projectId,projectNumber,name)"
```

If this issue affects your account, you may see hidden project IDs like:

```text
sys-01219650269145611594194724
sys-16811870700924204845773924
sys-98775854701091241033213827
```

These hidden projects often come from internal Google services such as Apps Script or Workspace-related automation.

To list only the hidden `sys-*` projects:

```bash
gcloud projects list --format="value(projectId)" | grep '^sys-' || true
```

Save the list in a variable:

```bash
SYS_PROJECTS="$(gcloud projects list --format='value(projectId)' | grep '^sys-' || true)"
printf '%s\n' "$SYS_PROJECTS"
```

---

## Step 7: Delete Hidden `sys-*` Projects

### Option A: Bulk-delete every `sys-*` project currently visible

```bash
gcloud projects list --format="value(projectId)" | grep '^sys-' | xargs -r -n1 gcloud projects delete --quiet
```

### Option B: Delete them one by one and show progress

```bash
for P in $(gcloud projects list --format="value(projectId)" | grep '^sys-' || true); do
  echo "Deleting $P"
  gcloud projects delete "$P" --quiet || echo "FAILED: $P"
done
```

### Option C: Delete a specific pasted list

```bash
printf '%s\n' \
sys-01219650269145611594194724 \
sys-16811870700924204845773924 \
sys-26048002610255661296327049 \
sys-61569674775294031274343499 \
sys-72665791778780126972202244 \
sys-75430492299389994388625318 \
sys-82364015101639467583370405 \
sys-84661668686776700986199020 \
sys-98775854701091241033213827 \
| xargs -n1 gcloud projects delete --quiet
```

---

## Step 8: Verify Project Deletion

Check what remains:

```bash
gcloud projects list --format="table(projectId,projectNumber,name)"
```

If you see:

```text
Listed 0 items.
```

that means the current project list is empty.

If some still remain, inspect their lifecycle state:

```bash
for P in $(gcloud projects list --format="value(projectId)" || true); do
  echo -n "$P: "
  gcloud projects describe "$P" --format="value(lifecycleState)"
done
```

A state of `DELETE_REQUESTED` usually means the deletion has been accepted and is in progress.

---

## Step 9: If Project Deletion Fails, Fix Permissions

If you get an error like:

```text
The caller does not have permission
```

check your roles:

```bash
gcloud organizations get-iam-policy "$ORG_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$ADMIN_EMAIL" \
  --format="table(bindings.role)"
```

If `roles/resourcemanager.projectDeleter` is missing, add it:

```bash
gcloud organizations add-iam-policy-binding "$ORG_ID" \
  --member="user:$ADMIN_EMAIL" \
  --role="roles/resourcemanager.projectDeleter"
```

Then retry the project deletion.

---

## Step 10: Find Hidden Folders That May Not Appear in the UI

Even after all `sys-*` projects are gone, hidden folders can still remain and still block Workspace deletion.

List top-level folders under the organization:

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

You may see something like:

```text
DISPLAY_NAME: system-gsuite
PARENT_NAME: organizations/286803580596
ID: 101752671225
```

Save that folder ID:

```bash
export SYS_FOLDER_ID="101752671225"
```

Now list folders inside it:

```bash
gcloud resource-manager folders list --folder="$SYS_FOLDER_ID"
```

You may see:

```text
DISPLAY_NAME: apps-script
PARENT_NAME: folders/101752671225
ID: 167668596053
```

Save that too:

```bash
export APPS_SCRIPT_FOLDER_ID="167668596053"
```

This is a very common hidden structure:

```text
Organization
└── system-gsuite
    └── apps-script
```

These folders often do **not** appear in the normal UI tree.

---

## Step 11: Check Hidden Folders for Remaining Projects

Check the top-level hidden folder:

```bash
gcloud projects list --filter="parent.id=$SYS_FOLDER_ID" --format="table(projectId,projectNumber,name)"
```

Check the nested folder:

```bash
gcloud projects list --filter="parent.id=$APPS_SCRIPT_FOLDER_ID" --format="table(projectId,projectNumber,name)"
```

If projects appear, delete them:

```bash
gcloud projects list --filter="parent.id=$APPS_SCRIPT_FOLDER_ID" --format="value(projectId)" \
  | xargs -r -n1 gcloud projects delete --quiet
```

---

## Step 12: Delete Hidden Folders Bottom-Up

Delete the innermost child folder first.

Delete the nested `apps-script` folder:

```bash
gcloud resource-manager folders delete "$APPS_SCRIPT_FOLDER_ID"
```

Verify its state:

```bash
gcloud resource-manager folders describe "$APPS_SCRIPT_FOLDER_ID"
```

Then delete the parent `system-gsuite` folder:

```bash
gcloud resource-manager folders delete "$SYS_FOLDER_ID"
```

Verify its state:

```bash
gcloud resource-manager folders describe "$SYS_FOLDER_ID"
```

If everything worked, the folders should move into `DELETE_REQUESTED`.

Now verify there are no remaining folders:

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

> **Important**
>
> Hidden folders must be deleted **from the bottom up**.  
> Do not try to delete `system-gsuite` before deleting `apps-script` if `apps-script` still exists.

---

## Step 13: If Folder Deletion Fails, Fix Folder Permissions

If folder deletion fails with an error like:

```text
The caller does not have permission to access folders instance ...
```

you are likely missing `roles/resourcemanager.folderAdmin`.

Add it:

```bash
gcloud organizations add-iam-policy-binding "$ORG_ID" \
  --member="user:$ADMIN_EMAIL" \
  --role="roles/resourcemanager.folderAdmin"
```

Verify again:

```bash
gcloud organizations get-iam-policy "$ORG_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$ADMIN_EMAIL" \
  --format="table(bindings.role)"
```

Then retry the folder deletions.

---

## Step 14: Check for Liens

If something still refuses to delete even though the permissions look correct, check for liens.

### Organization-level liens

```bash
gcloud alpha resource-manager liens list --organization="$ORG_ID"
```

### Folder-level liens

```bash
gcloud alpha resource-manager liens list --folder="$SYS_FOLDER_ID"
```

or:

```bash
gcloud alpha resource-manager liens list --folder="$APPS_SCRIPT_FOLDER_ID"
```

### Project-level liens

```bash
gcloud alpha resource-manager liens list --project="PROJECT_ID_HERE"
```

If a lien is returned, delete it using the lien name:

```bash
gcloud alpha resource-manager liens delete "LIEN_NAME_HERE"
```

---

## Step 15: Deep Scan for Remaining Assets

If:

- `gcloud projects list` shows no projects
- the UI still looks empty
- Google Admin still says resources remain

do a broader search:

```bash
gcloud asset search-all-resources \
  --scope="organizations/$ORG_ID" \
  --format="table(name,assetType)"
```

If you want to inspect a specific folder:

```bash
gcloud asset search-all-resources \
  --scope="folders/$SYS_FOLDER_ID" \
  --format="table(name,assetType)"
```

Anything still listed here is a candidate blocker.

---

## Step 16: Final Console Verification

Before going back to Google Admin, verify everything from the console.

### No projects remain

```bash
gcloud projects list
```

### No hidden folders remain

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

### No obvious remaining assets remain

```bash
gcloud asset search-all-resources \
  --scope="organizations/$ORG_ID" \
  --format="table(name,assetType)"
```

---

## Step 17: Delete the Workspace / Admin Account

Once the hidden projects and folders are gone:

1. Go to:
   - `https://admin.google.com/`
2. Sign in as the Workspace admin
3. Navigate to:
   - **Account**
   - **Account settings**
   - **Delete account**
4. Retry the deletion

At this point, the Google Cloud resource blocker should be gone.

> **Important**
>
> You are not manually deleting the organization node here.  
> Once the real Cloud blockers are gone and Workspace deletion succeeds, Google removes the organization automatically.

---

## Troubleshooting

### Error: `core/project` is set to a project number

Symptom:

```text
The value of `core/project` property is set to project number...
```

Fix:

```bash
gcloud config unset project
```

---

### Error: `The caller does not have permission` when deleting a project

You likely lack:

- `roles/resourcemanager.projectDeleter`

Fix:

```bash
gcloud organizations add-iam-policy-binding "$ORG_ID" \
  --member="user:$ADMIN_EMAIL" \
  --role="roles/resourcemanager.projectDeleter"
```

---

### Error: `The caller does not have permission` when deleting a folder

You likely lack:

- `roles/resourcemanager.folderAdmin`

Fix:

```bash
gcloud organizations add-iam-policy-binding "$ORG_ID" \
  --member="user:$ADMIN_EMAIL" \
  --role="roles/resourcemanager.folderAdmin"
```

---

### The UI shows no child resources or no folder dropdown

That does **not** mean the org is empty.

Check from Cloud Shell:

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

and then, for any folder you find:

```bash
gcloud resource-manager folders list --folder="FOLDER_ID"
```

---

### `gcloud projects list` shows 0 items, but Google Admin still says resources exist

That often means folders still exist, especially:

- `system-gsuite`
- `apps-script`

Check:

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

---

### I only know the project number, not the project ID

List both:

```bash
gcloud projects list --format="table(projectId,projectNumber,name)"
```

Deletion commands require the **project ID**, not the numeric project number.

---

### The shell prompt email does not match the account actually being used

Always trust:

```bash
gcloud config get-value account
```

Use that value for IAM bindings and verification.

---

## Quick Reference

### Set variables

```bash
export ADMIN_EMAIL="$(gcloud config get-value account)"
export ORG_ID="YOUR_ORG_ID_HERE"
```

### Clear bad default project config

```bash
gcloud config unset project
```

### Check your roles

```bash
gcloud organizations get-iam-policy "$ORG_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$ADMIN_EMAIL" \
  --format="table(bindings.role)"
```

### Grant all needed roles

```bash
for ROLE in \
  roles/resourcemanager.organizationAdmin \
  roles/resourcemanager.folderViewer \
  roles/resourcemanager.projectDeleter \
  roles/resourcemanager.folderAdmin
do
  gcloud organizations add-iam-policy-binding "$ORG_ID" \
    --member="user:$ADMIN_EMAIL" \
    --role="$ROLE"
done
```

### List hidden `sys-*` projects

```bash
gcloud projects list --format="value(projectId)" | grep '^sys-' || true
```

### Delete hidden `sys-*` projects

```bash
gcloud projects list --format="value(projectId)" | grep '^sys-' | xargs -r -n1 gcloud projects delete --quiet
```

### List folders under the organization

```bash
gcloud resource-manager folders list --organization="$ORG_ID"
```

### List child folders under a folder

```bash
gcloud resource-manager folders list --folder="FOLDER_ID"
```

### Delete a folder

```bash
gcloud resource-manager folders delete "FOLDER_ID"
```

### Check liens

```bash
gcloud alpha resource-manager liens list --organization="$ORG_ID"
```

### Deep scan remaining assets

```bash
gcloud asset search-all-resources \
  --scope="organizations/$ORG_ID" \
  --format="table(name,assetType)"
```

---

## Summary

If Google Admin says you still have Google Cloud resources but the Google Cloud UI looks empty, the blockers are often hidden internal resources such as:

- hidden `sys-*` projects
- hidden `system-gsuite` folders
- hidden `apps-script` folders

The reliable cleanup flow is:

1. Open Cloud Shell
2. Clear bad `gcloud` config
3. Find the organization ID and active admin email
4. Assign all required organization-level roles
5. Delete all hidden `sys-*` projects
6. Find and delete hidden folders bottom-up
7. Check for liens and remaining assets if needed
8. Retry deletion in Google Admin

That is the cleanup pattern that resolves the circular “delete Google Cloud resources first” blocker when the UI alone is not enough.

