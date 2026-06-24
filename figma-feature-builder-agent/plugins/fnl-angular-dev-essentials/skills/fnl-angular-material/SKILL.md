---
name: fnl-angular-material
description: 'fnl-angular-material.'
---

# Reactive Forms (Angular 21+)

Reactive Forms only — **never** template-driven (`[(ngModel)]`). Forms are typed, validated in the class, and (when their data is app state) bridged to NgRx.

## Use When

Use this skill whenever you build or fix a form — declaring typed `FormControl`/`FormGroup`/`FormArray`, attaching sync/async/cross-field validators, displaying `mat-error`s, reacting to `valueChanges`, building dynamic field lists, or bridging a form to the NgRx store. Do **not** use it to create the NgRx slice itself (`fnl-ngrx-state-management`), to style a Material form-field's appearance (`fnl-angular-material`), for general component/template authoring (`fnl-component-authoring`), or for test authoring (`fnl-angular-testing`).

## ⛔ Non-Negotiable Rules (violations are bugs)

1. **Reactive Forms only** — `FormControl` / `FormGroup` / `FormArray`; never `ngModel` on any control.
2. **Strictly typed** — `new FormControl<string>('', { nonNullable: true })`; never an untyped or `any` control.
3. **Validators in the class**, attached at control creation — never validation logic in the template.
4. **Bind with `[formControl]` / `formControlName`** — never `[(ngModel)]`.
5. **`ReactiveFormsModule` imported** in any component using a form control.
6. **FormControls are a dispatch bridge, not the source of truth** when the data is app state — restore from the store on init with `{ emitEvent: false }`, push changes via `valueChanges` + `dispatch` (see NgRx skill).
7. **Unsubscribe `valueChanges`** with `takeUntilDestroyed(this.destroyRef)` — never leak a subscription.
8. **No `any`** — type every control, group value, and validator return.

## Typed Controls & Groups

```typescript
import { FormControl, FormGroup, FormArray, Validators } from '@angular/forms';

readonly searchCtrl = new FormControl<string>('', { nonNullable: true });

readonly form = new FormGroup({
  name:  new FormControl<string>('', { nonNullable: true, validators: [Validators.required] }),
  email: new FormControl<string>('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
  age:   new FormControl<number | null>(null, { validators: [Validators.min(18)] }),
});
```

- `nonNullable: true` keeps the type non-null and resets to the initial value (not `null`).
- Prefer the explicit constructor; `FormBuilder` (`this.fb.group({...})` via `inject(FormBuilder)`) is acceptable for large groups.
- Read typed values with `this.form.getRawValue()` (includes disabled controls) or `this.form.value`.

## FormArray — Dynamic Field Lists

```typescript
readonly phones = new FormArray<FormControl<string>>([]);

protected addPhone(): void {
  this.phones.push(new FormControl<string>('', { nonNullable: true, validators: [Validators.required] }));
}
protected removePhone(index: number): void {
  this.phones.removeAt(index);
}
```
```html
@for (ctrl of phones.controls; track $index) {
  <mat-form-field appearance="outline">
    <input matInput [formControl]="ctrl" />
  </mat-form-field>
}
```

Use `FormArray` for any repeatable group of controls; track by `$index` when there is no stable id.

## Validators

| Need | Use |
|---|---|
| Built-ins | `Validators.required`, `email`, `min`, `max`, `minLength`, `maxLength`, `pattern` |
| One custom field rule | a `ValidatorFn` returning `ValidationErrors \| null` |
| Rule across two fields | a `ValidatorFn` on the **parent `FormGroup`** |
| Server/remote check | an `AsyncValidatorFn` returning `Observable<ValidationErrors \| null>` |

### Custom sync validator

```typescript
export function noWhitespace(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null =>
    (control.value ?? '').trim().length === 0 ? { whitespace: true } : null;
}
```

### Cross-field validator (on the group)

```typescript
export function passwordsMatch(): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const pw = group.get('password')?.value;
    const confirm = group.get('confirm')?.value;
    return pw === confirm ? null : { passwordsMismatch: true };
  };
}

readonly form = new FormGroup({
  password: new FormControl('', { nonNullable: true }),
  confirm:  new FormControl('', { nonNullable: true }),
}, { validators: passwordsMatch() });
```

### Async validator

```typescript
export function uniqueEmail(service: FeatureService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> =>
    service.isEmailTaken(control.value).pipe(
      map((taken) => (taken ? { emailTaken: true } : null)),
      catchError(() => of(null)),     // never block the form on a check failure
    );
}
// new FormControl('', { validators: [Validators.email], asyncValidators: [uniqueEmail(svc)], updateOn: 'blur' })
```

Use `updateOn: 'blur'` for async validators to avoid a request per keystroke.

## Displaying Errors (Material)

```html
<mat-form-field appearance="outline">
  <mat-label>Email</mat-label>
  <input matInput [formControl]="form.controls.email" />
  @if (form.controls.email.hasError('required')) { <mat-error>Email is required</mat-error> }
  @if (form.controls.email.hasError('email'))    { <mat-error>Enter a valid email</mat-error> }
  @if (form.controls.email.hasError('emailTaken')) { <mat-error>That email is taken</mat-error> }
</mat-form-field>
```

- `appearance="outline"` on every `mat-form-field` (per the angular-material skill).
- Show a control's errors after the user interacts — Material gates `mat-error` on touched/dirty automatically.
- Group-level errors (cross-field) render via the group: `@if (form.hasError('passwordsMismatch'))`.

## Reacting to Changes

```typescript
public constructor(private readonly destroyRef: DestroyRef) {}

public ngOnInit(): void {
  this.searchCtrl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    takeUntilDestroyed(this.destroyRef),
  ).subscribe((value) => this.store.dispatch(FeatureActions.filterUpdated({ filter: { query: value } })));
}
```

- Debounce text inputs (`debounceTime` + `distinctUntilChanged`) before acting.
- Restore a control from store state without re-emitting: `this.searchCtrl.setValue(value, { emitEvent: false })`.

## Submit

```typescript
protected onSubmit(): void {
  if (this.form.invalid) {
    this.form.markAllAsTouched();   // reveal all errors
    return;
  }
  this.store.dispatch(FeatureActions.saveRequested({ record: this.form.getRawValue() }));
}
```

- Guard on `form.invalid`; `markAllAsTouched()` to surface every error.
- Dispatch the submit — **never call the service directly from the component** (the effect owns the save; see the NgRx skill, Save triplet + `exhaustMap`).
- Disable the submit button while saving: `[disabled]="form.invalid || (isSaving() )"`.

## Checklist

- [ ] Reactive Forms only; `ReactiveFormsModule` imported; no `ngModel`
- [ ] Every control typed (`FormControl<T>`), `nonNullable` where appropriate; no `any`
- [ ] Validators attached in the class; cross-field on the group; async uses `updateOn: 'blur'`
- [ ] Errors shown with `@if (ctrl.hasError('x'))` + `<mat-error>`; `appearance="outline"`
- [ ] `valueChanges` debounced and unsubscribed via `takeUntilDestroyed`
- [ ] Form data that is app state is bridged to NgRx (restore with `{ emitEvent: false }`, dispatch changes)
- [ ] Submit guards `form.invalid`, `markAllAsTouched()`, dispatches (never calls the service directly)
