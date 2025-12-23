# Detail Pages Components

This document describes the detail page components created for the CRM suite, following the design from the first image and maintaining consistent theming across all detail pages.

## Components Overview

### Small Reusable Components (packages/ui/src/components/ui/)

#### New Components Created:
- **card.tsx** - Card container with header, content, and footer
- **progress.tsx** - Progress bar component
- **select.tsx** - Select dropdown component
- **textarea.tsx** - Multi-line text input component
- **checkbox.tsx** - Checkbox input component
- **detail-page.tsx** - Specialized components for detail pages:
  - `DetailPageHeader` - Header with title, status badge, and action buttons
  - `DetailCard` - Card wrapper for detail sections
  - `ActivityItem` - Timeline item for activities
  - `LeadScore` - Lead scoring component with progress bar
  - `QuickAction` - Quick action button component
- **info-field.tsx** - Information field display components:
  - `InfoField` - Individual field with label and value
  - `InfoGrid` - Grid layout for multiple fields

### Main Detail Page Components (apps/web/components/)

#### 1. LeadDetailPage
**File:** `lead-detail-page.tsx`

**Features:**
- Header with lead name, status badge, and action buttons (Claim, Convert, Edit, Delete)
- Lead details card with company, assigned to, email, phone, source, and created date
- Activity timeline with filterable activities
- Sidebar with:
  - Accounts/Contacts section with create/link account options
  - Lead score with progress bar
  - Quick actions (Send Email, Send WhatsApp)

**Props:**
```typescript
interface LeadDetailPageProps {
  lead: Lead
  onBack?: () => void
  onEdit?: () => void
  onDelete?: () => void
  onConvert?: () => void
  onSendEmail?: () => void
  onSendWhatsApp?: () => void
  onCreateAccount?: () => void
  onLinkAccount?: () => void
}
```

#### 2. AccountDetailPage
**File:** `account-detail-page.tsx`

**Features:**
- Header with account name and action buttons (Edit, Delete)
- Basic information bar with industry, website, account owner, and phone
- Tabbed interface with "Related" and "Details" tabs
- Related tab shows:
  - Contacts table with add contact functionality
  - Opportunities table with add opportunity functionality
- Details tab shows:
  - Address information (billing and shipping)
  - Additional information (description, revenue, company size)
  - System information (account owner, created/updated by, status)
  - Quick actions sidebar
- Edit mode with save/cancel buttons

**Props:**
```typescript
interface AccountDetailPageProps {
  account: Account
  contacts?: Contact[]
  opportunities?: Opportunity[]
  onBack?: () => void
  onEdit?: () => void
  onDelete?: () => void
  onSendEmail?: () => void
  onSendWhatsApp?: () => void
  onScheduleMeeting?: () => void
  onAddContact?: () => void
  onAddOpportunity?: () => void
  onReassign?: () => void
  onSave?: () => void
  onCancel?: () => void
}
```

#### 3. ContactDetailPage
**File:** `contact-detail-page.tsx`

**Features:**
- Header with contact name and action buttons (Upload, Download, Email, Edit)
- Basic information bar with account name, position, email, and phone
- Tabbed interface with "Related" and "Details" tabs
- Details tab shows:
  - Address information (mailing address, city, state, zip, country)
  - Additional information (description, LinkedIn, preferred contact method, alternate email, time zone)
  - System information (created/updated by, contact status)
  - Quick actions sidebar
- Edit mode with save/cancel buttons

**Props:**
```typescript
interface ContactDetailPageProps {
  contact: Contact
  onBack?: () => void
  onEdit?: () => void
  onDelete?: () => void
  onSendEmail?: () => void
  onSendWhatsApp?: () => void
  onScheduleMeeting?: () => void
  onUpload?: () => void
  onDownload?: () => void
  onSave?: () => void
  onCancel?: () => void
}
```

## Design Theme

All components follow the design theme from the first image:

### Color Palette:
- **Background:** Light gray (`bg-gray-50`)
- **Cards:** White background with subtle shadows
- **Text:** Dark gray for primary text, muted gray for secondary text
- **Borders:** Light gray borders with rounded corners
- **Status Badges:** Color-coded based on status (New, Active, etc.)

### Layout:
- **Grid System:** Responsive grid layout (1 column on mobile, 2-3 columns on desktop)
- **Spacing:** Consistent padding and margins throughout
- **Cards:** Rounded corners with subtle shadows
- **Buttons:** Rounded corners with appropriate variants (primary, outline, ghost)

### Typography:
- **Headings:** Bold, larger font sizes for hierarchy
- **Body Text:** Clean, readable font sizes
- **Labels:** Smaller, muted text for field labels

## Usage Example

```tsx
import { LeadDetailPage } from "@/components/lead-detail-page"

const lead = {
  id: "1",
  name: "Michael Anderson",
  company: "Innovate Tech Solutions LLC",
  email: "michael.anderson@innovatetech.com",
  phone: "+1 (555) 234-5678",
  source: "Website Contact Form",
  assignedTo: "Sarah Johnson",
  createdAt: "08 Oct, 2025 02:15 PM",
  status: "New",
  score: 75,
  activities: [
    {
      id: "1",
      title: "Form Submitted",
      description: "Submitted contact form in email.",
      time: "4 hours ago"
    }
  ]
}

function MyLeadPage() {
  return (
    <LeadDetailPage
      lead={lead}
      onBack={() => router.back()}
      onEdit={() => setEditing(true)}
      onDelete={() => deleteLead(lead.id)}
      onConvert={() => convertLead(lead.id)}
      onSendEmail={() => sendEmail(lead.email)}
      onSendWhatsApp={() => sendWhatsApp(lead.phone)}
    />
  )
}
```

## Integration

The detail pages are now fully integrated into the Lead Management Dashboard. Users can:

1. **Click on any lead** in the Lead Master, Assigned Leads, or Unassigned Leads tabs to view the lead detail page
2. **Click on any account** in the Accounts tab to view the account detail page  
3. **Click on any contact** in the Contacts tab to view the contact detail page
4. **Use the back button** on any detail page to return to the dashboard

The detail pages are accessible through the "View Details" action in the dropdown menu for each table row, or by clicking the action buttons directly.

## Integration Notes

1. **Icons:** All components use Lucide React icons for consistency
2. **Responsive:** All components are responsive and work on mobile and desktop
3. **Accessibility:** Components include proper ARIA labels and keyboard navigation
4. **Theming:** Components use the existing design system and can be easily customized
5. **State Management:** Components are controlled and expect parent components to manage state

## Future Enhancements

- Add loading states for async operations
- Add error handling and validation
- Add more customization options for styling
- Add keyboard shortcuts for common actions
- Add drag-and-drop functionality for reordering items
