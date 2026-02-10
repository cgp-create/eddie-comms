# Permission Issue with media/inbound Files

**Date:** 2026-02-09 13:52 UTC  
**Status:** Blocking PDF extraction

---

## ✅ Good News

pdfplumber is now installed and working:
```bash
python3 -c "import pdfplumber; print('pdfplumber available')"
→ pdfplumber available
```

---

## ❌ Bad News

Cannot access files in `/workspace/media/inbound/`:

```bash
id
→ uid=0(root) gid=0(root)

ls -la /workspace/media/inbound/*.pdf | head -3
→ -rw------- 1 1000 1000 40324 Feb 9 file_28...pdf
→ -rw------- 1 1000 1000 40324 Feb 9 file_23...pdf

python3 -c "import pdfplumber; pdfplumber.open('/workspace/media/inbound/file_28...pdf')"
→ PermissionError: [Errno 13] Permission denied

chmod 644 /workspace/media/inbound/file_28...pdf
→ chmod: Operation not permitted
```

---

## Root Cause

Files in `media/inbound/` are:
- Owned by uid 1000 (openclaw user on host)
- Mode 600 (owner-only read/write)
- Volume-mounted from host
- Container root cannot access due to volume security boundary

---

## Fix Needed

**Option A:** Run PDF extraction as uid 1000 instead of root

**Option B:** Fix file permissions on host side when files are created

**Option C:** Mount `/workspace/media/inbound` with different permissions

**Option D:** Copy files to a location Eddie can access before processing

---

## Test Case

Once fixed, test with:
```bash
python3 /home/openclaw/homebrew/pdf_extract.py extract /workspace/media/inbound/file_28---9869f464-87b1-4e55-9d49-99b075474711.pdf
```

Should extract 16 pages of text from Carlos's ReportLab PDF.

---

pdfplumber install: ✅  
File access: ❌
