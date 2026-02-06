# Deployment Instructions - Folder Structure Preservation Fix

## Summary of Changes Applied

All code changes from `docs/FIX_APPLIED_SUMMARY.md` have been applied with additional bug fixes:

### Files Modified:

1. **lambda/split_pdf/main.py**
   - Added folder structure preservation logic
   - Fixed folder_path extraction for files without subfolders
   - Added debug logging

2. **lambda/accessibility_checker_before_remidiation/main.py**
   - Updated to use original_pdf_key for downloads
   - Added folder_path support
   - Added fallback logic for missing original_pdf_key

3. **lambda/add_title/myapp.py**
   - Added folder_path extraction from merged_file_key
   - Updated to handle Java Lambda output wrapper
   - Fixed folder path preservation in result files

4. **lambda/accessability_checker_after_remidiation/main.py**
   - Added folder_path extraction from save_path
   - Updated save_to_s3 to preserve folder structure

5. **docker_autotag/autotag.py**
   - Updated to use S3_CHUNK_KEY environment variable
   - Fixed S3 key parsing to preserve folder structure

6. **javascript_docker/alt-text.js**
   - Fixed variable scoping bug (fileDirectory and fileKey)
   - Updated to use S3_CHUNK_KEY
   - Fixed all S3 paths to preserve folder structure

7. **lambda/java_lambda/PDFMergerLambda/src/main/java/com/example/App.java**
   - Preserved folder structure in merged PDF output path

8. **app.py** (CDK Configuration)
   - Added result_path to ECS Task 1 to preserve input
   - Fixed ECS Task 2 environment variable passing
   - Added result_selector to Java Lambda task
   - Fixed add_title Lambda payload passing

## Key Bug Fixes

### Bug 1: Folder Path Extraction
**Issue**: Files uploaded to `pdf/document.pdf` (no subfolder) were incorrectly extracting folder_path as "document" instead of empty string.

**Fix**: Updated logic in `split_pdf/main.py` to properly handle both cases:
- `pdf/batch1/doc.pdf` → folder_path = `batch1`
- `pdf/doc.pdf` → folder_path = `` (empty)

### Bug 2: Missing original_pdf_key Fallback
**Issue**: If original_pdf_key wasn't passed through the pipeline, the Lambda would fail.

**Fix**: Added fallback logic to construct original_pdf_key from file_basename if not provided.

### Bug 3: JavaScript Variable Scoping
**Issue**: `fileDirectory` and `fileKey` were undefined in `modifyPDF` function.

**Fix**: Added these as parameters to the function signature.

## Expected Folder Structure

### Input:
```
pdf/
├── document.pdf                    (root level)
└── Sample-PDFs/
    └── Sample-Syllabus-1.pdf      (subfolder)
```

### Output:
```
temp/
├── document/                       (root level - no subfolder prefix)
│   ├── document_chunk_1.pdf
│   ├── output_autotag/
│   ├── FINAL_document_chunk_1.pdf
│   ├── merged_document.pdf
│   └── accessability-report/
└── Sample-PDFs/                    (subfolder preserved)
    └── Sample-Syllabus-1/
        ├── Sample-Syllabus-1_chunk_1.pdf
        ├── output_autotag/
        ├── FINAL_Sample-Syllabus-1_chunk_1.pdf
        ├── merged_Sample-Syllabus-1.pdf
        └── accessability-report/

result/
├── COMPLIANT_document.pdf          (root level)
└── Sample-PDFs/                    (subfolder preserved)
    └── COMPLIANT_Sample-Syllabus-1.pdf
```

## Deployment Steps

### Step 1: Commit Changes

```bash
# Check what files were modified
git status

# Stage all modified files
git add lambda/split_pdf/main.py
git add lambda/accessibility_checker_before_remidiation/main.py
git add lambda/add_title/myapp.py
git add lambda/accessability_checker_after_remidiation/main.py
git add docker_autotag/autotag.py
git add javascript_docker/alt-text.js
git add lambda/java_lambda/PDFMergerLambda/src/main/java/com/example/App.java
git add app.py

# Commit with descriptive message
git commit -m "Add folder structure preservation and bug fixes

- Enhancement 1: Support folder uploads (preserve original PDF path)
- Enhancement 2: Preserve folder structure in all outputs
- Fix: Folder path extraction for root-level files
- Fix: JavaScript variable scoping in alt-text.js
- Fix: Java Lambda output wrapping
- Fix: ECS task state preservation in Step Functions
- Add: Debug logging for troubleshooting"

# Push to your repository
git push origin main
```

### Step 2: Deploy via Deployment Script

```bash
# Run the deployment script
./deploy.sh

# When prompted, select the PDF-to-PDF solution
```

### Step 3: Alternative - Deploy via CDK Directly

```bash
# Deploy the stack
cdk deploy PDFAccessibility --require-approval never
```

### Step 4: Verify Deployment

After deployment, check that:

1. All Lambda functions are updated (check "Last modified" timestamp in AWS Console)
2. ECS task definitions have new revisions
3. Step Functions state machine is updated

## Testing

### Test Case 1: Root Level File
```bash
# Upload a file to root pdf/ folder
aws s3 cp test.pdf s3://YOUR-BUCKET/pdf/test.pdf

# Expected output:
# - temp/test/test_chunk_1.pdf
# - result/COMPLIANT_test.pdf
```

### Test Case 2: Single Subfolder
```bash
# Upload a file to a subfolder
aws s3 cp test.pdf s3://YOUR-BUCKET/pdf/batch1/test.pdf

# Expected output:
# - temp/batch1/test/test_chunk_1.pdf
# - result/batch1/COMPLIANT_test.pdf
```

### Test Case 3: Nested Subfolders
```bash
# Upload a file to nested subfolders
aws s3 cp test.pdf s3://YOUR-BUCKET/pdf/2024/january/test.pdf

# Expected output:
# - temp/2024/january/test/test_chunk_1.pdf
# - result/2024/january/COMPLIANT_test.pdf
```

## Troubleshooting

### Issue: Lambda still using old code
**Symptom**: Error shows old code paths (e.g., `f"pdf/{file_key}"`)

**Solution**: 
1. Verify deployment completed successfully
2. Check Lambda function "Last modified" timestamp
3. Manually update Lambda if needed:
   ```bash
   cd lambda/accessibility_checker_before_remidiation
   zip -r function.zip .
   aws lambda update-function-code --function-name YOUR-FUNCTION-NAME --zip-file fileb://function.zip
   ```

### Issue: 404 Not Found errors
**Symptom**: `ClientError: An error occurred (404) when calling the HeadObject operation`

**Solution**:
1. Check CloudWatch logs for the split_pdf Lambda to see what paths it's creating
2. Look for the DEBUG log line: `DEBUG split_pdf: original_key=..., file_basename=..., folder_path=...`
3. Verify the original_pdf_key is being passed through the Step Functions

### Issue: Files in wrong location
**Symptom**: Files appear in `temp/filename/` instead of `temp/folder/filename/`

**Solution**:
1. Check the split_pdf Lambda logs for the folder_path value
2. Verify the input file was uploaded to the correct location (must be under `pdf/`)
3. Check that the Step Functions execution input includes `original_pdf_key` and `folder_path`

## Rollback Instructions

If you need to rollback:

```bash
# Revert the commit
git revert HEAD

# Push the revert
git push origin main

# Redeploy
./deploy.sh
```

## Support

For issues or questions:
1. Check CloudWatch Logs for each Lambda function
2. Review Step Functions execution history
3. Verify S3 bucket structure matches expected patterns
4. Check that all environment variables are set correctly in ECS tasks
