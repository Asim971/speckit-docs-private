# ⚡ DEVELOPMENT AGENT V2 - CRITICAL TASK ASSIGNMENT

**Handoff ID**: `handoff_20251018_002`  
**Priority**: 🔴 **CRITICAL**  
**Time Budget**: 30 minutes  
**Status**: Awaiting Execution

---

## 🎯 Your Mission

Fix the **Dashboard fetch loop** causing 20x API calls instead of 1, blocking deployment.

---

## 📍 What You Need to Know

### The Problem
```
Dashboard component fetches /api/v1/patients/patient-001/stats
Expected: 1 fetch per session
Actual:   20 fetches per session (1900% regression)
Impact:   DEPLOYMENT BLOCKED
```

### Evidence
- **Test Report**: `TESTING_AGENT_V1_PHASE2_FINAL_REPORT.md` (Test #3)
- **Request Trace**: 20 sequential GET calls within 3-second window
- **Component**: `apps/refill-portal/src/components/Dashboard.tsx`

### Root Cause (Hypothesis)
React component rendering 20 times due to:
- Missing useEffect dependency array
- State update cycle triggering multiple renders
- React Query misconfiguration

---

## 🔧 Your Action Items

### STEP 1: Locate & Inspect Dashboard Component
```bash
# Navigate to component
cd apps/refill-portal
cat src/components/Dashboard.tsx

# Look for:
# - useEffect hooks without dependency arrays
# - State updates in render (bad pattern)
# - React Query hook configuration
```

**Acceptance**: Identified all useEffect hooks and their dependencies

---

### STEP 2: Identify the Culprit
Look for patterns like:
```typescript
// ❌ BAD: Missing dependency array = runs every render
useEffect(() => {
  fetchStats();
}, []); // Missing deps!

// ❌ BAD: Empty deps but state changed in effect
useEffect(() => {
  setData(newData);
  // triggers re-render, which triggers effect again = loop
}, []);

// ❌ BAD: useQuery without cache optimization
const { data } = useQuery({
  queryKey: ['stats'],
  queryFn: fetchStats,
  staleTime: 0, // Refetches on every mount!
});
```

**Acceptance**: Pinpointed exact useEffect causing 20 fetches

---

### STEP 3: Apply Fixes
```typescript
// ✅ GOOD: Complete dependency array
useEffect(() => {
  fetchStats();
}, [userId]); // Include dependencies!

// ✅ GOOD: Query cache optimization
const { data } = useQuery({
  queryKey: ['stats', userId],
  queryFn: fetchStats,
  staleTime: 5 * 60 * 1000, // 5 min cache
  gcTime: 10 * 60 * 1000,   // 10 min garbage collection
});

// ✅ GOOD: Deduplication with request key
const { data } = useQuery({
  queryKey: ['stats', userId],
  queryFn: fetchStats,
  staleTime: Infinity, // Don't refetch if in cache
});
```

**Acceptance**: All useEffect hooks have complete dependency arrays

---

### STEP 4: Verify Locally
```bash
# Install dependencies
npm install

# Build the app
npm run build

# Start dev server
npm run dev -- --host --port 4173

# In browser console:
# 1. Open Network tab (Filter: XHR)
# 2. Navigate to Dashboard
# 3. Should see: 1 GET /api/v1/patients/patient-001/stats
# 4. NOT: 20 GET requests in 3 seconds
```

**Acceptance**: Browser Network tab shows exactly 1 fetch to `/stats`

---

### STEP 5: Add Regression Guard Test
Create or update test to prevent 20x multiplier in future:

```typescript
// File: e2e/single-fetch-prevention-test.spec.ts
test('Dashboard fetches stats endpoint exactly once', async ({ page }) => {
  let statsFetchCount = 0;
  
  page.on('response', (response) => {
    if (response.url().includes('/api/v1/patients/patient-001/stats')) {
      statsFetchCount++;
    }
  });
  
  await page.goto('http://localhost:3000/dashboard');
  await page.waitForLoadState('networkidle');
  
  // Assert exactly 1 fetch
  expect(statsFetchCount).toBe(1);
});
```

**Acceptance**: New test passes and prevents future regressions

---

## 📋 Files to Edit

### Primary File
```
apps/refill-portal/src/components/Dashboard.tsx
- Check all useEffect hooks
- Verify dependency arrays
- Optimize React Query cache
```

### Secondary Files (if needed)
```
apps/refill-portal/src/hooks/usePatientStats.ts
- Verify hook implementation
- Check cache configuration

apps/refill-portal/src/queries/patientQueries.ts
- Verify query options
- Check staleTime/gcTime settings
```

---

## ✅ Success Criteria

- [x] Identified root cause (useEffect/React Query issue)
- [ ] Fixed dependency arrays
- [ ] Verified single fetch in browser Network tab
- [ ] Added regression guard test
- [ ] **Testing Agent v1.0 re-runs Test #3: PASSES**
- [ ] Deployment unblocked

---

## ⏱️ Timeline

| Task | Duration |
|------|----------|
| Locate + inspect | 5 min |
| Identify culprit | 5 min |
| Apply fixes | 10 min |
| Verify locally | 5 min |
| Add regression guard | 5 min |
| **Total** | **30 min** |

**ETA**: 04:52 UTC

---

## 🔄 What Happens Next

1. ✅ You: Fix Dashboard component
2. 📝 You: Document changes + reasoning
3. 🔄 **Testing Agent v1.0**: Re-runs Test #3 (15 min)
4. ✅ Orchestrator: Validates quality gate pass
5. 🚀 **Next Phase**: Phase 3 Advanced Testing or Phase 4 Sign-Off

---

## 📎 Related Documents

- **Orchestration Summary**: `ORCHESTRATOR_PHASE2_QUALITY_GATE_REMEDIATION.md`
- **Testing Report**: `TESTING_AGENT_V1_PHASE2_FINAL_REPORT.md`
- **Testing Decision**: `TESTING_AGENT_V1_PHASE2_DECISION.md`
- **Handoff Details**: `.speckit/state/orchestration/handoffs/handoff_20251018_002.json`

---

## 🆘 Need Help?

**Question**: "Where exactly is Dashboard.tsx?"  
**Answer**: `apps/refill-portal/src/components/Dashboard.tsx`

**Question**: "How do I know if the fix works?"  
**Answer**: Browser Network tab should show 1 GET /stats (not 20)

**Question**: "What if I can't find the issue?"  
**Answer**: Escalate to Senior Developer + Architecture Review

---

**Status**: 🟢 **READY FOR EXECUTION**  
**Authority**: Orchestrator Agent (orch_20251018_002)  
**Approval**: Testing Agent v1.0 (deployment blocking authority)

**Execute this task immediately. Production deployment is blocked.**
