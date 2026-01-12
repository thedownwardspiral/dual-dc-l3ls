# Pre-Commit Security Checklist

Complete this checklist before pushing to GitHub.

## ✅ Files Verified

### Protected Files (Gitignored)
- [x] `deploy.yml` - Contains CloudVision token (VERIFIED: gitignored)
- [x] `group_vars/DC-L3LS-FABRIC.yml` - Contains passwords (VERIFIED: gitignored)
- [x] `config_backup/` - Contains device backups (VERIFIED: gitignored)

### Template Files (Safe to Commit)
- [x] `deploy.yml.example` - Placeholders only (VERIFIED: no credentials)
- [x] `group_vars/DC-L3LS-FABRIC.yml.example` - Placeholders only (VERIFIED: no credentials)

### Documentation Files
- [x] `SECURITY_REVIEW.md` - No actual credentials (VERIFIED: sanitized)
- [x] `SETUP.md` - Instructions only (VERIFIED: safe)
- [x] `CLAUDE.md` - No credentials (VERIFIED: safe)

## ⚠️ Action Items

### Before First Push
- [ ] Review: Has this repo been initialized with credentials before?
  - If YES: Create a fresh repository instead
  - If NO: Proceed with push

### After Any Credential Exposure
- [ ] Revoke CloudVision API token in CV portal
- [ ] Change switch admin password on all devices
- [ ] Update BGP passwords on all devices

## 🔍 Final Verification Commands

Run these commands to verify no credentials will be committed:

```bash
# Check what will be committed
git status

# Verify sensitive files are ignored
git check-ignore deploy.yml group_vars/DC-L3LS-FABRIC.yml

# Search for potential credentials (should return 0 or only gitignored files)
git ls-files | xargs grep -l "cv_token\|ansible_password" | grep -v ".example"

# Dry-run to see what would be added
git add -n .
```

## ✅ Safe to Push Indicators

You're safe to push when:
1. `git check-ignore` confirms both sensitive files are ignored
2. `git status` does NOT show `deploy.yml` or `DC-L3LS-FABRIC.yml` as staged
3. Template files (`.example`) are staged/committed
4. No credentials appear in files tracked by git

## 🚨 Emergency: Credentials Were Committed

If credentials were accidentally committed:

### Option 1: Fresh Start (Recommended)
```bash
cd ..
mv dual-dc-l3ls dual-dc-l3ls-old
mkdir dual-dc-l3ls
cd dual-dc-l3ls
git init
# Copy files from old directory, excluding .git
```

### Option 2: Clean Git History (Advanced)
```bash
# Use BFG Repo Cleaner (recommended)
bfg --delete-files deploy.yml --no-blob-protection
bfg --delete-files DC-L3LS-FABRIC.yml --no-blob-protection
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Then rotate ALL credentials immediately
```

## 📝 Notes

- Git history NEVER forgets - deleted files remain in history
- .gitignore only prevents NEW files from being tracked
- If unsure, create a fresh repository
- Always rotate credentials if exposure is suspected
