# Forms and Validation

## Reactive Forms, siempre

`FormBuilder` + `FormGroup` + `Validators` para todo formulario de
alta/edición. `[(ngModel)]`/`FormsModule` solo se acepta para estado de UI
suelto (ej. un input de búsqueda en una tabla) — nunca para un formulario
de negocio.

```ts
private readonly _fb = inject(FormBuilder);

form = this._fb.group({
  nombre: ['', Validators.required],
  email: ['', [Validators.required, Validators.email]],
  precio: [0, [Validators.required, Validators.min(0)]],
});
```

No hay `Validators` custom en uso real — solo los validators built-in de
Angular (`required`, `min`, `max`, `minLength`, `maxLength`, `email`,
`pattern`). No inventes una fábrica de validators custom salvo que el
usuario la pida explícitamente.

## Helpers de validación (`shared/helpers/form-helpers.ts`)

Funciones puras, no un servicio — se importan directo en el componente:

```ts
export function hasFormError(form: FormGroup, field: string): boolean {
  const control = form.get(field);
  return !!control && control.invalid && control.touched;
}

export function getFormError(form: FormGroup, field: string): string {
  const control = form.get(field);
  return control ? getControlError(control) : '';
}

export function getControlError(control: AbstractControl): string {
  if (control.hasError('required')) return 'Este campo es obligatorio.';
  if (control.hasError('email')) return 'Correo inválido.';
  if (control.hasError('minlength')) return `Mínimo ${control.getError('minlength').requiredLength} caracteres.`;
  if (control.hasError('maxlength')) return `Máximo ${control.getError('maxlength').requiredLength} caracteres.`;
  return 'Campo inválido.';
}
```

Uso en template: `@if (hasFormError(form, 'email')) { <small>{{ getFormError(form, 'email') }}</small> }`.

## Envío del formulario — patrón obligatorio de 2 líneas

Antes de chequear `form.invalid`, siempre:

```ts
submit(): void {
  this.form.markAllAsTouched();
  Object.values(this.form.controls).forEach(c => c.markAsDirty());
  if (this.form.invalid) return;
  // ... continuar con el submit
}
```

Las dos líneas son necesarias por separado: PrimeNG solo pinta el borde
rojo de inválido con la combinación de clases CSS `.ng-invalid.ng-dirty`,
mientras que `hasFormError()` de arriba chequea `.touched`. Omitir
cualquiera de las dos líneas rompe el feedback visual o el mensaje de
error (o ambos).

## Inputs — siempre un componente/directiva de PrimeNG

Nunca un `<input>`/`<select>` nativo en un formulario — usa
`pInputText`/`p-select`/`p-inputnumber`/`p-datepicker`/`p-checkbox`/
`p-toggleswitch`/`pTextarea`/`p-inputmask` según el tipo de dato. Un input
nativo pierde el estilo de error de PrimeNG en inválido, aunque el resto
del formulario luzca correcto.

## Campos de fecha

Un campo de fecha se parchea al formulario como objeto `Date` (para
`p-datepicker`) y se convierte a string ISO (`YYYY-MM-DD`) recién al
enviar. La lista de qué campos son fecha vive como constante exportada
junto al modelo, y se recorre tanto al parchear como al enviar:

```ts
// junto al modelo
export const PEDIDO_DATE_FIELDS = ['fecha_entrega', 'fecha_pago'] as const;

// al parchear (edición)
PEDIDO_DATE_FIELDS.forEach(f => { if (pedido[f]) form.get(f)?.setValue(new Date(pedido[f])); });

// al enviar
PEDIDO_DATE_FIELDS.forEach(f => {
  const v = form.get(f)?.value;
  if (v instanceof Date) form.get(f)?.setValue(v.toISOString().split('T')[0]);
});
```

No es un helper genérico reutilizable — es una lista puntual por entidad,
a propósito, junto al modelo que describe.
