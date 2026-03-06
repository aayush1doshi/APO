# Technical Specification / Functional Specification
## ATPCAT Filter Enhancement for Pendo Orders Processing

---

### Document Information

| Field | Details |
|-------|---------|
| **Document Type** | Technical Specification / Functional Specification |
| **Module** | SCM Redeployment - Pendo Processing |
| **Function Module** | `ZSCM_REDEP_FETCH_PENDO` |
| **Date** | Current Date |
| **Status** | Implemented |

---

## 1. Executive Summary

This document describes the enhancement made to the function module `ZSCM_REDEP_FETCH_PENDO` to filter orders based on ATPCAT (Available-to-Promise Category) before processing them through the sales order validation function module `Z_SCM_REDEP_FETCH_SO_PENDO`.

### 1.1 Objective

To optimize the processing of Pendo orders by:
- Processing only orders with `ATPCAT = 'BM'` through the sales order validation function
- Preserving all other orders (non-BM) directly in the output without validation
- Maintaining existing deletion logic for BM orders only

---

## 2. Business Requirement

### 2.1 Current Behavior

Previously, all orders from `et_requirements` were:
1. Passed to `Z_SCM_REDEP_FETCH_SO_PENDO` for validation
2. Subjected to deletion logic if not found in the validation results
3. Deleted from `et_requirements` if validation failed

### 2.2 Required Behavior

The new requirement specifies:
1. **Only orders with `ATPCAT = 'BM'`** should be passed to `Z_SCM_REDEP_FETCH_SO_PENDO`
2. **Deletion logic should be applied only to BM orders** (orders not found in validation results should be removed)
3. **All other orders (ATPCAT ≠ 'BM')** should remain in `et_requirements` without any processing or deletion

### 2.3 Business Justification

- Reduces unnecessary processing overhead for non-BM orders
- Maintains data integrity for BM orders through validation
- Preserves all non-BM orders in the output for downstream processing

---

## 3. Technical Specification

### 3.1 Function Module Details

**Function Module:** `ZSCM_REDEP_FETCH_PENDO`

**Location:** Lines 2957-3000

### 3.2 Data Structures

#### 3.2.1 Internal Tables

| Table | Type | Description |
|-------|------|-------------|
| `lt_vbeln` | `TABLE OF lty_vbeln` | Contains sales order numbers (only BM orders) |
| `lt_lifsk` | `TABLE OF lty_lifsk` | Contains validation results from `Z_SCM_REDEP_FETCH_SO_PENDO` |
| `et_requirements` | `ZBAPI10501SLREQOUT_T` | Output table containing all requirements (includes both BM and non-BM) |

#### 3.2.2 Type Definitions

```abap
TYPES: BEGIN OF lty_vbeln,
  vbeln TYPE vbeln_va,
END OF lty_vbeln.

TYPES: BEGIN OF lty_lifsk,
  vbeln TYPE vbeln_va,
  lifsk TYPE lifsk,
END OF lty_lifsk.
```

### 3.3 Logic Flow

#### 3.3.1 Step 1: Filter Orders by ATPCAT

```abap
LOOP AT et_requirements INTO lw_requirements.
  IF lw_requirements-atpcat = 'BM'.
    lw_vbeln-vbeln = lw_requirements-order_number.
    APPEND lw_vbeln TO lt_vbeln.
    CLEAR: lw_vbeln.
  ENDIF.
  CLEAR: lw_requirements.
ENDLOOP.
```

**Purpose:** Build `lt_vbeln` containing only orders with `ATPCAT = 'BM'`

#### 3.3.2 Step 2: Call Validation Function (Only for BM Orders)

```abap
IF lt_vbeln IS NOT INITIAL.
  CALL FUNCTION 'Z_SCM_REDEP_FETCH_SO_PENDO' DESTINATION lw_rfc-rfcdest
    EXPORTING
      it_vbeln  = lt_vbeln
    IMPORTING
      et_so_det = lt_lifsk
      et_return = lt_return4.
ENDIF.
```

**Purpose:** Validate only BM orders through RFC call

#### 3.3.3 Step 3: Apply Deletion Logic (Only for BM Orders)

```abap
LOOP AT et_requirements ASSIGNING <lfs_requirements>.
  IF <lfs_requirements> IS ASSIGNED.
    IF <lfs_requirements>-atpcat = 'BM'.
      CLEAR: lw_lifsk.
      READ TABLE lt_lifsk INTO lw_lifsk WITH KEY vbeln = <lfs_requirements>-order_number.
      IF sy-subrc <> 0.
        <lfs_requirements>-order_number = ' '.
      ENDIF.
    ENDIF.
  ENDIF.
ENDLOOP.

DELETE et_requirements WHERE order_number IS INITIAL AND atpcat = 'BM'.
```

**Purpose:** 
- Check if BM orders exist in validation results
- Blank out `order_number` for BM orders not found in validation
- Delete only BM orders with blank `order_number`

#### 3.3.4 Step 4: Preserve Non-BM Orders

Non-BM orders remain in `et_requirements` without any processing.

---

## 4. Code Changes

### 4.1 Modified Section

**File:** `Redeployment ATD.txt`  
**Function Module:** `ZSCM_REDEP_FETCH_PENDO`  
**Lines:** 2957-3000

### 4.2 Before (Old Logic)

```abap
IF et_requirements IS NOT INITIAL.
  CLEAR: lw_requirements.
  LOOP AT et_requirements INTO lw_requirements.
    lw_vbeln-vbeln = lw_requirements-order_number.
    APPEND lw_vbeln TO lt_vbeln.
    CLEAR: lw_vbeln.
    CLEAR: lw_requirements.
  ENDLOOP.

  REFRESH: lt_lifsk, lt_return4.
  CALL FUNCTION 'Z_SCM_REDEP_FETCH_SO_PENDO' DESTINATION lw_rfc-rfcdest
    EXPORTING
      it_vbeln  = lt_vbeln
    IMPORTING
      et_so_det = lt_lifsk
      et_return = lt_return4.

  LOOP AT et_requirements ASSIGNING <lfs_requirements>.
    IF <lfs_requirements> IS ASSIGNED.
      CLEAR: lw_lifsk.
      READ TABLE lt_lifsk INTO lw_lifsk WITH KEY vbeln = <lfs_requirements>-order_number.
      IF sy-subrc <> 0.
        <lfs_requirements>-order_number = ' '.
      ENDIF.
    ENDIF.
  ENDLOOP.

  DELETE et_requirements WHERE order_number IS INITIAL.
ENDIF.
```

### 4.3 After (New Logic)

```abap
IF et_requirements IS NOT INITIAL.
  CLEAR: lw_requirements.
  REFRESH: lt_vbeln.
* Build lt_vbeln only for orders with ATPCAT = 'BM'
  LOOP AT et_requirements INTO lw_requirements.
    IF lw_requirements-atpcat = 'BM'.
      lw_vbeln-vbeln = lw_requirements-order_number.
      APPEND lw_vbeln TO lt_vbeln.
      CLEAR: lw_vbeln.
    ENDIF.
    CLEAR: lw_requirements.
  ENDLOOP.

* Only call Z_SCM_REDEP_FETCH_SO_PENDO for orders with ATPCAT = 'BM'
  IF lt_vbeln IS NOT INITIAL.
    REFRESH: lt_lifsk, lt_return4.
    CALL FUNCTION 'Z_SCM_REDEP_FETCH_SO_PENDO' DESTINATION lw_rfc-rfcdest
      EXPORTING
        it_vbeln  = lt_vbeln
      IMPORTING
        et_so_det = lt_lifsk
        et_return = lt_return4.

* Apply deletion logic only for orders with ATPCAT = 'BM'
    LOOP AT et_requirements ASSIGNING <lfs_requirements>.
      IF <lfs_requirements> IS ASSIGNED.
* Only process orders with ATPCAT = 'BM'
        IF <lfs_requirements>-atpcat = 'BM'.
          CLEAR: lw_lifsk.
          READ TABLE lt_lifsk INTO lw_lifsk WITH KEY vbeln = <lfs_requirements>-order_number.
          IF sy-subrc <> 0.
            <lfs_requirements>-order_number = ' '.
          ENDIF.
        ENDIF.
      ENDIF.
    ENDLOOP.

* Delete only BM orders that were not found in lt_lifsk
    DELETE et_requirements WHERE order_number IS INITIAL AND atpcat = 'BM'.
  ENDIF.

* Orders with ATPCAT <> 'BM' remain in et_requirements (no processing needed)
ENDIF.
```

### 4.4 Key Changes Summary

| Change | Description |
|--------|-------------|
| **Filter Addition** | Added `IF lw_requirements-atpcat = 'BM'` condition before adding to `lt_vbeln` |
| **Conditional RFC Call** | Wrapped RFC call in `IF lt_vbeln IS NOT INITIAL` check |
| **Selective Deletion Logic** | Added `IF <lfs_requirements>-atpcat = 'BM'` check before applying deletion logic |
| **Selective DELETE Statement** | Changed `DELETE et_requirements WHERE order_number IS INITIAL` to `DELETE et_requirements WHERE order_number IS INITIAL AND atpcat = 'BM'` |

---

## 5. Functional Specification

### 5.1 Input Parameters

No changes to input parameters. The function module continues to accept the same input parameters.

### 5.2 Output Parameters

**`et_requirements`** - Modified behavior:
- **BM Orders:** Only validated BM orders that pass validation remain
- **Non-BM Orders:** All non-BM orders remain without validation

### 5.3 Processing Rules

| Order Type | Processing | Validation | Deletion Logic | Final Status |
|------------|------------|------------|----------------|--------------|
| **ATPCAT = 'BM'** | ✓ Processed | ✓ Validated via RFC | ✓ Applied | Removed if validation fails |
| **ATPCAT ≠ 'BM'** | ✗ Not processed | ✗ Not validated | ✗ Not applied | Always preserved |

### 5.4 Business Rules

1. **BM Orders (ATPCAT = 'BM'):**
   - Must be validated through `Z_SCM_REDEP_FETCH_SO_PENDO`
   - Must exist in validation results (`lt_lifsk`) to be retained
   - If not found in validation results, order is deleted from output

2. **Non-BM Orders (ATPCAT ≠ 'BM'):**
   - Skip validation process
   - Always retained in output
   - No processing overhead

---

## 6. Impact Analysis

### 6.1 Affected Components

| Component | Impact | Description |
|-----------|--------|-------------|
| `ZSCM_REDEP_FETCH_PENDO` | **High** | Core logic modified |
| `Z_SCM_REDEP_FETCH_SO_PENDO` | **Medium** | Called only for BM orders (reduced load) |
| Downstream processes using `et_requirements` | **Low** | Output structure unchanged, but content may differ |

### 6.2 Performance Impact

**Positive Impacts:**
- Reduced RFC calls (only for BM orders)
- Reduced processing time for non-BM orders
- Lower network traffic

**Considerations:**
- Performance improvement depends on ratio of BM to non-BM orders
- If most orders are non-BM, significant performance gain expected

### 6.3 Data Impact

- **BM Orders:** Only validated orders remain (same as before)
- **Non-BM Orders:** All orders remain (new behavior - previously some may have been deleted)

---

## 7. Testing Strategy

### 7.1 Unit Testing Scenarios

#### Test Case 1: Only BM Orders
**Input:** `et_requirements` contains only orders with `ATPCAT = 'BM'`  
**Expected:** All orders processed through validation, deletion logic applied

#### Test Case 2: Only Non-BM Orders
**Input:** `et_requirements` contains only orders with `ATPCAT ≠ 'BM'`  
**Expected:** No RFC call made, all orders preserved in output

#### Test Case 3: Mixed Orders
**Input:** `et_requirements` contains both BM and non-BM orders  
**Expected:** 
- BM orders: Validated and filtered
- Non-BM orders: All preserved

#### Test Case 4: BM Orders - Validation Failure
**Input:** BM orders that fail validation (not in `lt_lifsk`)  
**Expected:** Failed BM orders deleted from output

#### Test Case 5: BM Orders - Validation Success
**Input:** BM orders that pass validation (found in `lt_lifsk`)  
**Expected:** Validated BM orders retained in output

### 7.2 Integration Testing

1. Test with calling function module `ZSCM_REDEP_CALC_ATD`
2. Verify downstream processing with mixed order types
3. Validate RFC connectivity and error handling

### 7.3 Regression Testing

- Verify existing functionality for BM orders remains intact
- Ensure no impact on other ATPCAT values
- Validate error handling and return messages

---

## 8. Error Handling

### 8.1 RFC Call Errors

If `Z_SCM_REDEP_FETCH_SO_PENDO` fails:
- Error messages returned in `lt_return4`
- All BM orders will be deleted (as `lt_lifsk` will be empty)
- Non-BM orders remain unaffected

### 8.2 Data Integrity

- Empty `et_requirements`: No processing, no errors
- Empty `lt_vbeln`: No RFC call made, all orders preserved
- Invalid ATPCAT values: Treated as non-BM (preserved)

---

## 9. Deployment Considerations

### 9.1 Pre-Deployment

- [ ] Code review completed
- [ ] Unit tests executed
- [ ] Integration tests completed
- [ ] Performance impact assessed
- [ ] Documentation updated

### 9.2 Deployment Steps

1. Backup existing function module
2. Transport the modified code
3. Activate function module
4. Monitor initial runs for errors
5. Verify output data integrity

### 9.3 Post-Deployment

- Monitor RFC call volumes
- Verify output data correctness
- Check performance metrics
- Collect user feedback

---

## 10. Rollback Plan

If issues are encountered:

1. **Immediate Rollback:** Revert to previous version of function module
2. **Data Impact:** Non-BM orders may have been preserved (new behavior) - assess if this causes issues
3. **Communication:** Notify stakeholders of rollback

---

## 11. Appendix

### 11.1 Related Function Modules

- `ZSCM_REDEP_FETCH_PENDO` - Main function module (modified)
- `Z_SCM_REDEP_FETCH_SO_PENDO` - Validation function module (called)
- `ZSCM_REDEP_CALC_ATD` - Calling function module

### 11.2 Related Tables/Structures

- `ZBAPI10501SLREQOUT_T` - Requirements output table
- `ZREDEP_VBELN_T` - Sales order number table
- `ZREDEP_LIFSK_DET_T` - Delivery block details table

### 11.3 Constants

- `ATPCAT = 'BM'` - ATP Category for orders requiring validation

---

## 12. Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Current Date | Development Team | Initial implementation - ATPCAT filter enhancement |

---

## 13. Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Technical Lead | | | |
| Functional Lead | | | |
| Project Manager | | | |

---

**End of Document**

