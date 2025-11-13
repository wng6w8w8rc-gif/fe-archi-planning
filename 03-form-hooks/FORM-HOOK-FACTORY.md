# Form Hook Factory Pattern

## Summary

This refactoring creates a factory function to standardize form hook creation across the codebase. Multiple form hooks follow the same pattern using `react-hook-form` with `zod` validation, but each is implemented separately with boilerplate code.

### Files Affected

**New Files:**

- `lib/forms/createFormHook.ts` - Factory function

**Modified Files:**

- `store/auth/forms/useContactForm.ts`
- `store/auth/forms/useAddressForm.ts`
- `store/auth/forms/useSignUpForm.ts`
- `store/auth/forms/useEmailLoginForm.ts`
- `store/auth/forms/useLoginForm.ts`
- `store/auth/forms/usePostalCodeForm.ts`
- `store/feedback/useFeedbackForm.tsx`
- `store/report-issue/forms/useReportIssueForm.tsx`
- `containers/booking-info/useBookingInfoForm.tsx`
- `containers/booking/services/elder-care-booking/elder-care-form-modal/useElderCareForm.tsx`
- `containers/profile/service-profile/service-profile-form/elder/useElderCareForm.tsx`
- `containers/profile/service-profile/service-profile-form/baby/useBabyProfileForm.tsx`
- `containers/profile/service-profile/service-profile-form/pet/usePetProfileForm.tsx`
- Other form hooks following the same pattern

### Key Changes

- **Factory function** `createFormHook` standardizes form hook creation
- **Configuration-based approach** with schema and default values
- **Consistent form patterns** across all forms
- **Reduced boilerplate** for new form hooks

### Benefits

- **Consistent form patterns** across the codebase
- **Reduced boilerplate** - less code to write for new forms
- **Easier maintenance** - form logic in one place
- **Type safety** maintained with generics
- **Standardized validation** behavior

## Implementation Realization

### Before: Duplicate Form Hook Pattern

**Multiple form hooks with similar structure**

```typescript:luce-fe/src/store/auth/forms/useContactForm.ts
import type { UseFormReturn } from "react-hook-form";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
// ... imports

const contactSchema = z.object({
  // ... schema definition
});

export type ContactFormData = z.input<typeof contactSchema>;

export interface ContactFormReturn extends UseFormReturn<ContactFormData> {
  defaultValues: ContactFormData;
}

export const initialContactFormValues: ContactFormData = {
  // ... default values
};

export const useContactForm = (
  initialValues?: ContactFormData,
): ContactFormReturn => ({
  defaultValues: initialValues || initialContactFormValues,
  ...useForm<ContactFormData>({
    mode: "onChange",
    defaultValues: initialValues,
    resolver: zodResolver(contactSchema),
  }),
});
```

```typescript:luce-fe/src/store/auth/forms/useAddressForm.ts
// Identical pattern - 70+ lines
// Different schema and default values
```

```typescript:luce-fe/src/store/feedback/useFeedbackForm.tsx
// Identical pattern - 30+ lines
// Different schema and default values
```

**Pattern repeated in:**

- `useSignUpForm.ts`
- `useEmailLoginForm.ts`
- `useLoginForm.ts`
- `useReportIssueForm.tsx`
- `useBookingInfoForm.tsx`
- And more...

### After: Factory Function

**createFormHook.ts** - Factory function

```typescript:luce-fe/src/lib/forms/createFormHook.ts
import type { UseFormReturn, UseFormProps } from "react-hook-form";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import type { z } from "zod";

type FormHookConfig<T extends z.ZodType> = {
  schema: T;
  defaultValues: z.input<T>;
  formOptions?: Omit<UseFormProps<z.input<T>>, "resolver" | "defaultValues">;
};

export type FormHookReturn<T extends z.ZodType> = UseFormReturn<z.input<T>> & {
  defaultValues: z.input<T>;
};

export function createFormHook<T extends z.ZodType>(
  config: FormHookConfig<T>,
) {
  return (initialValues?: z.input<T>): FormHookReturn<T> => {
    const values = initialValues || config.defaultValues;

    return {
      defaultValues: values,
      ...useForm<z.input<T>>({
        mode: "onChange",
        defaultValues: values,
        resolver: zodResolver(config.schema),
        ...config.formOptions,
      }),
    };
  };
}
```

**Updated form hooks** - Simple configuration

```typescript:luce-fe/src/store/auth/forms/useContactForm.ts
import { z } from "zod";
import { createFormHook } from "@/lib/forms/createFormHook";
import { superRefinePhoneNumber } from "@/lib/helpers/phone-number-validation";
import { nameRE } from "@/constants/regex";
import { __ } from "@/lib/i18n";

const contactSchema = z.object({
  id: z.string().optional(),
  firstName: z
    .string()
    .min(1, __("First name is required"))
    .refine((value) => nameRE.test(value), {
      error: __("Please avoid using special characters"),
    }),
  lastName: z
    .string()
    .min(1, __("Last name is required"))
    .refine((value) => nameRE.test(value), {
      error: __("Please avoid using special characters"),
    }),
  email: z
    .array(z.object({ value: z.string().email(__("Invalid email")) }))
    .optional(),
  phoneNumber: z
    .array(
      z
        .object({
          value: z.string(),
          nationCode: z.string(),
        })
        .superRefine((arg, ctx) => {
          superRefinePhoneNumber({
            code: arg.nationCode,
            phoneNumber: arg.value,
            ctx,
            phoneNumberFieldName: "value",
          });
        }),
    )
    .optional(),
  primary: z.boolean().optional(),
});

export type ContactFormData = z.input<typeof contactSchema>;

export interface ContactFormReturn extends FormHookReturn<typeof contactSchema> {}

export const initialContactFormValues: ContactFormData = {
  firstName: "",
  lastName: "",
  email: [{ value: "" }],
  phoneNumber: [{ value: "", nationCode: "SG/65" }],
  primary: true,
};

export const useContactForm = createFormHook({
  schema: contactSchema,
  defaultValues: initialContactFormValues,
});
```

**Usage remains the same**

```typescript
const form = useContactForm();
// or with initial values
const form = useContactForm({ firstName: "John", ... });
```

### Usage Examples

**Example 1: Basic Form Usage**

```typescript
import { View, Button, Input } from "@/components/ui";
import { InputFormControl } from "@/components/shared/input-form-control";
import { useContactForm } from "@/store/auth/forms/useContactForm";
import { useCreateContactStore } from "@/store/profile/createContact";
import { __ } from "@/lib/i18n";

function ContactForm() {
  const { control, handleSubmit } = useContactForm();
  const { fetch: createContact, loading } = useCreateContactStore();

  const onSubmit = handleSubmit(async (data) => {
    await createContact({
      requestPayload: {
        input: {
          firstName: data.firstName,
          lastName: data.lastName,
          // ... other fields
        },
      },
    });
  });

  return (
    <View>
      <InputFormControl
        name="firstName"
        label={__("First Name")}
        control={control}
        component={Input}
        componentProps={{
          placeholder: __("Your First Name"),
        }}
      />
      <InputFormControl
        name="lastName"
        label={__("Last Name")}
        control={control}
        component={Input}
        componentProps={{
          placeholder: __("Your Last Name"),
        }}
      />
      <Button variant="primary" onClick={onSubmit} loading={loading} children={__("Submit")} />
    </View>
  );
}
```

**Example 2: Form with Initial Values (Edit Mode)**

```typescript
import { View, Button } from "@/components/ui";
import { InputFormControl } from "@/components/shared/input-form-control";
import { useContactForm } from "@/store/auth/forms/useContactForm";
import { useUpdateContactStore } from "@/store/profile/updateContact";
import { useClientStore } from "@/store/auth/client";
import type { Contact } from "@/types/users";

function EditContactForm({ contact }: { contact: Contact }) {
  const { control, handleSubmit } = useContactForm({
    id: contact.id,
    firstName: contact.firstName,
    lastName: contact.lastName,
    email: contact.email?.map((email) => ({ value: email })),
    phoneNumber: contact.phoneNumber?.map((phone) => ({
      value: phone,
      nationCode: "SG/65",
    })),
  });

  const { fetch: updateContact, loading } = useUpdateContactStore();
  const { fetch: client } = useClientStore();

  const onSubmit = handleSubmit(async (data) => {
    await updateContact({
      requestPayload: {
        input: {
          id: contact.id,
          firstName: data.firstName,
          lastName: data.lastName,
          // ... other fields
        },
      },
    });
    // Refresh client data
    await client({ requestPayload: { id: contact.clientId } });
  });

  return (
    <View>
      <InputFormControl name="firstName" label={__("First Name")} control={control} component={Input} />
      <Button variant="primary" onClick={onSubmit} loading={loading} children={__("Update")} />
    </View>
  );
}
```

**Example 3: Creating a New Form Hook**

```typescript
// store/auth/forms/useNewForm.ts
import { z } from "zod";
import { createFormHook } from "@/lib/forms/createFormHook";
import { __ } from "@/lib/i18n";

const newFormSchema = z.object({
  name: z.string().min(1, __("Name is required")),
  email: z.string().email(__("Invalid email")),
});

export type NewFormData = z.input<typeof newFormSchema>;

export const useNewForm = createFormHook({
  schema: newFormSchema,
  defaultValues: {
    name: "",
    email: "",
  },
});

// Usage in component
function NewFormComponent() {
  const { control, handleSubmit } = useNewForm();

  const onSubmit = handleSubmit((data) => {
    console.log(data);
  });

  return (
    <View>
      <InputFormControl name="name" label={__("Name")} control={control} component={Input} />
      <Button onClick={onSubmit} children={__("Submit")} />
    </View>
  );
}
```

## Migration Summary

**Code Reduction**:

- Before: ~10 form hooks × ~70 lines average = ~700 lines
- After: 1 factory (~30 lines) + ~10 form hooks × ~50 lines = ~530 lines
- **Net reduction**: ~170 lines (24% reduction) + elimination of boilerplate

**Files Affected**:

1. ✅ `store/auth/forms/useContactForm.ts`
2. ✅ `store/auth/forms/useAddressForm.ts`
3. ✅ `store/auth/forms/useSignUpForm.ts`
4. ✅ `store/auth/forms/useEmailLoginForm.ts`
5. ✅ `store/auth/forms/useLoginForm.ts`
6. ✅ `store/feedback/useFeedbackForm.tsx`
7. ✅ `store/report-issue/forms/useReportIssueForm.tsx`
8. ✅ `containers/booking-info/useBookingInfoForm.tsx`
9. ✅ And other form hooks...

**Benefits**:

- ✅ Consistent form patterns across codebase
- ✅ Reduced boilerplate for new forms
- ✅ Easier to add validation rules globally
- ✅ Type-safe with generics
- ✅ Standardized form behavior

**Migration Strategy**:

1. Create `createFormHook` factory function
2. Migrate one form hook at a time (start with simplest)
3. Test each migration thoroughly
4. Update all form hook files
5. Consider adding common validation rules to factory

**Optional Enhancements**:

- Add common validation rules (email, phone, etc.) as factory options
- Add form mode configuration (onChange, onSubmit, etc.)
- Add custom resolver options
