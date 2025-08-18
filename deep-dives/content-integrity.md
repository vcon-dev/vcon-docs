# Content Integrity in vCon 0.3.0

## Overview
Content integrity is a core feature of vCon 0.3.0, providing cryptographic verification of conversation content.

## Implementation Details

### Content Hashing
```python
# Calculate hash for dialog content
dialog.calculate_content_hash()

# Verify content integrity
is_valid = dialog.verify_content_hash()
```

### Automatic Hash Calculation
Content hashes are automatically calculated when:
- Adding external files
- Modifying dialog content
- Updating party information

## Best Practices
- Always verify content hashes when loading external content
- Enable automatic hash calculation
- Implement hash verification in your workflow