# Example: Sign-Up Form

The sign-up form demonstrates the full stack in one component:
- Zod schema in `/lib/validations`
- React Hook Form with zodResolver
- All copy via `next-intl` (zero hardcoded strings)
- Tailwind v4 utilities only (no arbitrary values unless unavoidable)
- Inline field errors from RHF's `formState.errors`
- Unit test with Vitest + RTL
- Storybook story with autodocs

---

## src/lib/validations/auth.ts

```ts
import { z } from "zod";

export const signUpSchema = z.object({
  name: z.string().min(2, { message: "Name must be at least 2 characters" }),
  email: z.string().email({ message: "Enter a valid email address" }),
  password: z
    .string()
    .min(8, { message: "Password must be at least 8 characters" }),
});

export type SignUpInput = z.infer<typeof signUpSchema>;
```

---

## messages/en.json

Add these keys (merge with any existing content):

```json
{
  "SignUpForm": {
    "title": "Create an account",
    "nameLabel": "Full name",
    "namePlaceholder": "Jane Doe",
    "emailLabel": "Email address",
    "emailPlaceholder": "jane@example.com",
    "passwordLabel": "Password",
    "passwordPlaceholder": "Min. 8 characters",
    "submit": "Create account",
    "submitting": "Creating account…"
  }
}
```

## messages/\<locale\>.json

Add the same keys with placeholder translations (copy the English values and
mark them with a locale prefix so the developer knows they need real
translations):

```json
{
  "SignUpForm": {
    "title": "[<locale>] Create an account",
    "nameLabel": "[<locale>] Full name",
    "namePlaceholder": "[<locale>] Jane Doe",
    "emailLabel": "[<locale>] Email address",
    "emailPlaceholder": "[<locale>] jane@example.com",
    "passwordLabel": "[<locale>] Password",
    "passwordPlaceholder": "[<locale>] Min. 8 characters",
    "submit": "[<locale>] Create account",
    "submitting": "[<locale>] Creating account…"
  }
}
```

---

## src/components/ui/sign-up-form/SignUpForm.tsx

```tsx
"use client";

import { useTranslations } from "next-intl";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { signUpSchema, type SignUpInput } from "@/lib/validations/auth";

interface SignUpFormProps {
  onSubmit: (data: SignUpInput) => Promise<void>;
}

export function SignUpForm({ onSubmit }: SignUpFormProps) {
  const t = useTranslations("SignUpForm");

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<SignUpInput>({
    resolver: zodResolver(signUpSchema),
  });

  return (
    <form
      onSubmit={handleSubmit(onSubmit)}
      className="mx-auto w-full max-w-sm space-y-4"
      noValidate
    >
      <h1 className="text-2xl font-semibold">{t("title")}</h1>

      <div className="flex flex-col gap-1">
        <label htmlFor="name" className="text-sm font-medium">
          {t("nameLabel")}
        </label>
        <input
          id="name"
          type="text"
          autoComplete="name"
          placeholder={t("namePlaceholder")}
          className="rounded-md border px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
          {...register("name")}
        />
        {errors.name && (
          <p role="alert" className="text-sm text-red-600">
            {errors.name.message}
          </p>
        )}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="email" className="text-sm font-medium">
          {t("emailLabel")}
        </label>
        <input
          id="email"
          type="email"
          autoComplete="email"
          placeholder={t("emailPlaceholder")}
          className="rounded-md border px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
          {...register("email")}
        />
        {errors.email && (
          <p role="alert" className="text-sm text-red-600">
            {errors.email.message}
          </p>
        )}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="password" className="text-sm font-medium">
          {t("passwordLabel")}
        </label>
        <input
          id="password"
          type="password"
          autoComplete="new-password"
          placeholder={t("passwordPlaceholder")}
          className="rounded-md border px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
          {...register("password")}
        />
        {errors.password && (
          <p role="alert" className="text-sm text-red-600">
            {errors.password.message}
          </p>
        )}
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:opacity-50"
      >
        {isSubmitting ? t("submitting") : t("submit")}
      </button>
    </form>
  );
}
```

---

## src/components/ui/sign-up-form/SignUpForm.test.tsx

```tsx
import { screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { renderWithIntl } from "@/test-utils/intl";
import { SignUpForm } from "./SignUpForm";

const messages = {
  SignUpForm: {
    title: "Create an account",
    nameLabel: "Full name",
    namePlaceholder: "Jane Doe",
    emailLabel: "Email address",
    emailPlaceholder: "jane@example.com",
    passwordLabel: "Password",
    passwordPlaceholder: "Min. 8 characters",
    submit: "Create account",
    submitting: "Creating account…",
  },
};

function renderForm(onSubmit = vi.fn().mockResolvedValue(undefined)) {
  return renderWithIntl(<SignUpForm onSubmit={onSubmit} />, messages);
}

describe("SignUpForm", () => {
  it("renders all fields and submit button", () => {
    renderForm();
    expect(screen.getByLabelText(/full name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/email address/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /create account/i })).toBeInTheDocument();
  });

  it("shows validation errors when submitted empty", async () => {
    renderForm();
    await userEvent.click(screen.getByRole("button", { name: /create account/i }));
    await waitFor(() => {
      expect(screen.getAllByRole("alert").length).toBeGreaterThan(0);
    });
  });

  it("calls onSubmit with valid data", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);
    renderForm(onSubmit);

    await userEvent.type(screen.getByLabelText(/full name/i), "Jane Doe");
    await userEvent.type(screen.getByLabelText(/email address/i), "jane@example.com");
    await userEvent.type(screen.getByLabelText(/password/i), "password123");
    await userEvent.click(screen.getByRole("button", { name: /create account/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith(
        {
          name: "Jane Doe",
          email: "jane@example.com",
          password: "password123",
        },
        expect.anything() // RHF passes the submit event as second arg
      );
    });
  });
});
```

---

## src/components/ui/sign-up-form/SignUpForm.stories.tsx

```tsx
import type { Meta, StoryObj } from "@storybook/react";
import { withIntl } from "@/test-utils/intl";
import { SignUpForm } from "./SignUpForm";

const messages = {
  SignUpForm: {
    title: "Create an account",
    nameLabel: "Full name",
    namePlaceholder: "Jane Doe",
    emailLabel: "Email address",
    emailPlaceholder: "jane@example.com",
    passwordLabel: "Password",
    passwordPlaceholder: "Min. 8 characters",
    submit: "Create account",
    submitting: "Creating account…",
  },
};

const meta: Meta<typeof SignUpForm> = {
  component: SignUpForm,
  tags: ["autodocs"],
  decorators: [
    withIntl(messages),
    (Story) => (
      <div className="flex min-h-screen items-center justify-center p-8">
        <Story />
      </div>
    ),
  ],
  args: {
    onSubmit: async (data) => {
      await new Promise((r) => setTimeout(r, 1000));
      console.log("Submitted:", data);
    },
  },
};

export default meta;
type Story = StoryObj<typeof SignUpForm>;

export const Default: Story = {};
```
