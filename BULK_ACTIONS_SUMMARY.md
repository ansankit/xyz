# Bulk Actions for Accounts and Contacts

## Overview
This document outlines all the bulk actions available for accounts and contacts in the CRM system after selecting multiple items.

## Bulk Actions Available

### Accounts
When multiple accounts are selected, the following bulk actions are available:

1. **Export Selected** - Export only the selected accounts to a file (CSV/Excel format)
2. **Delete Selected** - Permanently delete all selected accounts (with confirmation dialog)

#### Available in UI:
- Checkboxes to select individual accounts or all accounts at once
- Selection counter showing how many accounts are selected
- Bulk action buttons appear when items are selected
- Confirmation dialog before deletion

#### Implementation Details:
- State managed via `selectedAccounts` in `lead-management-dashboard.tsx`
- Uses `AccountTable` component with `showCheckboxes` prop enabled
- Selection state passed through `selectedItems` and `onSelectionChange` props

---

### Contacts
When multiple contacts are selected, the following bulk actions are available:

1. **Export Selected** - Export only the selected contacts to a file (CSV/Excel format)
2. **Delete Selected** - Permanently delete all selected contacts (with confirmation dialog)

#### Available in UI:
- Checkboxes to select individual contacts or all contacts at once
- Selection counter showing how many contacts are selected
- Bulk action buttons appear when items are selected
- Confirmation dialog before deletion

#### Implementation Details:
- State managed via `selectedContacts` in `lead-management-dashboard.tsx`
- Uses `ContactTable` component with `showCheckboxes` prop enabled
- Selection state passed through `selectedItems` and `onSelectionChange` props

---

## Comparison with Leads Bulk Actions

### Leads have additional bulk actions:
1. **Convert Selected** - Convert multiple leads to contacts/accounts
2. **Assign** - Assign multiple leads to a sales person
3. **Claim** - Claim multiple unassigned leads
4. **Send to Email** - Send selected leads via email
5. **Delete Selected** - Delete selected leads
6. **Export** - Export leads to file

### Accounts & Contacts currently have:
1. **Export Selected** - Export selected items
2. **Delete Selected** - Delete selected items

---

## Future Enhancement Suggestions

### For Accounts:
- **Bulk Merge** - Merge multiple duplicate accounts
- **Bulk Assign Owner** - Assign account ownership to a sales person
- **Bulk Update Fields** - Update common fields (industry, status, etc.) for multiple accounts
- **Export to Campaign** - Add selected accounts to a campaign
- **Archive Selected** - Soft delete/archive accounts instead of permanent deletion

### For Contacts:
- **Bulk Merge** - Merge multiple duplicate contacts
- **Bulk Assign Owner** - Assign contact ownership to a sales person
- **Bulk Update Fields** - Update common fields (status, tags, etc.) for multiple contacts
- **Add to Campaign** - Add selected contacts to a campaign
- **Send Email** - Send bulk email to selected contacts
- **Archive Selected** - Soft delete/archive contacts instead of permanent deletion

---

## Technical Implementation

### Files Modified:
1. `apps/web/components/lead-management-dashboard.tsx`
   - Added state for `selectedAccounts` and `selectedContacts`
   - Added handlers for bulk operations
   - Added confirmation dialogs for delete operations
   
2. `apps/web/components/account-table.tsx`
   - Added checkbox support
   - Added props for selection management
   
3. `apps/web/components/contact-table.tsx`
   - Added checkbox support
   - Added props for selection management

### Components Used:
- `DataTable` component handles the checkbox UI
- `DeleteConfirmationDialog` for deletion confirmations
- `Button` components for action triggers

---

## API Endpoints Needed

Currently, the following single-item APIs exist:
- `DELETE /api/contacts/:id` - Delete single contact
- `DELETE /api/accounts/:id` - Delete single account (to be implemented)

**Recommended Bulk APIs to Implement:**
- `POST /api/accounts/delete-bulk` - Bulk delete accounts
- `POST /api/contacts/delete-bulk` - Bulk delete contacts
- `POST /api/accounts/export` - Export accounts
- `POST /api/contacts/export` - Export contacts
- `POST /api/accounts/update-bulk` - Bulk update accounts
- `POST /api/contacts/update-bulk` - Bulk update contacts

---

## User Experience Flow

1. User navigates to Accounts or Contacts tab
2. User selects one or more items using checkboxes
3. Selection count is displayed
4. Bulk action buttons appear
5. User clicks desired action
6. If destructive action (delete), confirmation dialog appears
7. User confirms action
8. Action is processed
9. Selection is cleared
10. Table is refreshed to show updated data

---

## Notes
- All bulk operations are currently processed item-by-item in the frontend
- This approach works for small selections but may need optimization for larger datasets
- Consider implementing server-side bulk operations for better performance
- Export functionality is currently a placeholder - implement actual CSV/Excel generation
