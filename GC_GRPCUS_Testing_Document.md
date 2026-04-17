# Testing Document: ZAPO_CVC_REALIGN (Rule 01 - GC-GRPCUS)

## 1. Objective
To verify that the newly deployed changes to the ABAP method `compare_cvc` within the class `lcl_realign` accurately align, correct, and update Group Customer (`p_grpcust`) allocations for mismatched records based on real-time sales assignments, while ignoring combinations that are already properly mapped.

## 2. Pre-requisites
- Access to the APO / test system containing the transport for the bug fix.
- Test Customer Validations setup across existing Sales Organizations (`p_saleorg`), Distribution Channels (`p_dc`), matching Division (`p_div`), and active Ship-To Parties (`p_eccshp`).
- At least one active record possessing an alignment mapping differing from the target relationship data (to prove realignment changes behavior).
- At least one active record natively matching the correct standard targets. 

## 3. Test Data
Below is the sample reference layout structure used to reproduce the baseline requirements.

| Prod. Allo | APO Produc | Sec | Division | Ship To Party | **Current Grp Cust** | Target Grp Cust | Remarks |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 24 | K6701 | P1 | 24 | 3000004103 | `5000002003` | `5000002003` | Record already correct |
| 24 | K6701 | P2 | 24 | 3000004103 | `5000004328` | `5000002003` | Record currently **wrong** |

<br>

## 4. Test Scenarios

### Test Case 1: Verification of Correct Group Customer Auto-Alignment
- **Scenario:** The target dataset holds an invalid existing group customer combination linked to valid hierarchical sales relationships (matching Sales Org, Division, Ship To).
- **Execution Steps:**
  1. Initialize target CVC characteristics.
  2. Map an invalid or obsolete Group Customer allocation (e.g., `5000004328`).
  3. Execute rule `GC-GRPCUS`.
- **Expected Result:** The `ZAPO_CVC_REALIGN` engine accurately sweeps `lt_tmp_knvp2`, identifies the mismatch against `5000004328`, loops through consecutive combinations properly, and overrides it to correctly propose / finalize `5000002003`.
- **Status:** [ ] Pass / [ ] Fail

### Test Case 2: Validation of Correct Group Customer Processing
- **Scenario:** The target dataset already has a valid prevailing group customer assigned.
- **Execution Steps:**
  1. Setup standard validation structures matching true target alignments.
  2. Verify target Group Customer correctly aligns natively on `5000002003`.
  3. Execute rule `GC-GRPCUS`.
- **Expected Result:** The engine smoothly runs the sequential internal validation (`sy-subrc = 0`), processes the matching fields accurately without erroring out, assesses the group customer as fully valid, and ensures no data overrides corrupt or alter it.
- **Status:** [ ] Pass / [ ] Fail

### Test Case 3: Empty Results Validation (Negative Scenario)
- **Scenario:** Characteristics fed inside do not natively reflect mapped customer data representations within relation data (`lt_knvp`).
- **Execution Steps:**
  1. Input arbitrary characteristic hierarchies mapping to a nonexistent dummy hierarchy.
  2. Execute rule `GC-GRPCUS`.
- **Expected Result:** The initial validation fails the ABAP `READ TABLE ... BINARY SEARCH` checks (`sy-subrc <> 0`). The inner iteration loop completely skips processing minimizing faulty behavior impacts resulting in zero empty updates. 
- **Status:** [ ] Pass / [ ] Fail

## 5. End Result Output & Validations
- After execution, navigate to the output logging framework for realignment actions. 
- Verify the status logs have updated appropriately indicating realigned variables versus bypassed targets.
- Validate via the internal tables/visual report output that target Group Customers consistently showcase uniform assignments regardless of erroneous original pre-processing statuses.

---
**Prepared By:** _______________  
**Date:** _______________  
**Sign Off:** _______________
