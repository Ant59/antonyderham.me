---
title: "Dealing with file inputs in Angular 6"
date: 2018-06-01
tags: [angular,form,input,file,directive,control,accessor,upload]
---

File inputs are strange. The value attribute on most inputs behaves as expected, giving the value of the field. HTML defines the `value` attribute on file inputs as "the path of the first file selected" ([see MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/file#Value)), not the file(s) itself. The selected files are found on the `files` attribute as a FileList. The leads to some issues with Angular when using forms. Angular has the concept of a `ControlValueAccessor` ([default implementation](https://github.com/angular/angular/blob/0cb4f12a7a57087ec4e8329a04d5dfc430764b45/packages/forms/src/directives/default_value_accessor.ts#L59)). The default accessor uses the `value` attribute. To get at the file itself, you'll need a custom value accessor.

The `ControlValueAccessor` is just an interface for a directive. You can create a new directive for accessing the file on a file input field and provide it with the `NG_VALUE_ACCESSOR` token.

{{< highlight typescript >}}
import { Directive, HostListener } from '@angular/core';
import { NG_VALUE_ACCESSOR, ControlValueAccessor } from '@angular/forms';

@Directive({
  selector: 'input[type=file]',
  providers: [
    { provide: NG_VALUE_ACCESSOR, useExisting: FileValueAccessor, multi: true }
  ]
})
export class FileValueAccessor implements ControlValueAccessor {
  @HostListener('change', ['$event.target.files']) onChange = (_: any) => { };
  @HostListener('blur') onTouched = () => { };

  writeValue(value: any) { }
  registerOnChange(fn: (_: any) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
}
{{</ highlight >}}

Once imported to your NgModule, file inputs will start providing the file object on their `files` attribute to Angular forms and the `valueChanges` items will be the selected file. The selector for the directive `input[type=file]` ensures this directive applies only to file input fields. The `'$event.target.files'` property selector in the change host listener specifies the `files` attribute value should be given when this event is emitted instead of the default.
