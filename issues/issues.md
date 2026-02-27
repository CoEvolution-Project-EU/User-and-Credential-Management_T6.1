# Issue Tracking Guidelines  
Component: User & Credential Management (UCM)  
Task ID: [TASK_ID]

---

## 1. When to Create an Issue

Create an issue for:

- Bug or malfunction
- Missing documentation
- Failed integration test
- Dependency blocker
- Feature request
- Security vulnerability
- Performance issue (>200ms latency)

---

## 2. Required Labels

bug  
enhancement  
documentation  
integration  
blocked  

---

## 3. Issue Template

All issues must follow this structure:

---

**Title**: [Short clear description]

**Description**:
- Summary of the problem or need
- Affected component: UCM / [TASK_ID]
- Steps to reproduce (if applicable)
- Expected outcome

**Status**: Open  
**Priority**: High / Medium / Low  
**Assigned to**: @username  
**Related milestone**: M18 / M20 / M30  
**Dependencies**: [List other components/issues]

---

## 4. Issue Lifecycle

1. Create issue with proper label
2. Assign owner
3. Link related merge request
4. Add debugging comments
5. Validate fix
6. Close only after confirmed test success

---

## 5. Integration-Related Issues

Must include:

- Affected external component
- API version used
- Token format (if auth-related)
- Environment details
- Logs (if applicable)

Label: integration

---

## 6. Blocked Issues

If blocked:

- Clearly state dependency
- Tag responsible partner
- Add expected resolution timeline

Label: blocked

---

## 7. Monitoring & Reporting

- Update issues weekly if active
- Reference milestone
- Ensure merge requests reference issue ID
- Close issues only after verification

---

## 8. Reporting to Project Level

Before milestone review:

- Export list of open issues
- Categorize by severity
- Confirm no High-priority issues remain unresolved
- Ensure documentation-related issues are closed

---

# Summary

Proper issue tracking ensures:

- Transparency
- Integration readiness
- Alignment with D3.2
- Milestone compliance
- Cross-partner coordination