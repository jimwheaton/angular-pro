## Angular Pro


## Advanced Components

#### Content projection with ng-content
- basic project content
```html
<employee-card>
  <label>Manager</label>
</employee-card>
```

```typescript
@Component({
  selector: 'employee-card',
  template: `
    <div class="card">
      <h3>Level</h3>
      <ng-content></ng-content>
  `
})
```
- content projection with projection slots
```html
<employee-card>
  <label>Manager</label>
  <button>Fire someone!</button>
</employee-card>
```

```typescript
@Component({
  selector: 'employee-card',
  template: `
    <div class="card">
      <h3>Level</h3>
      <ng-content select="label"></ng-content>
      <h3>Take action</h3>
      <ng-content select="button"></ng-content>
  `
})
```
- component projection with binding
```typescript
@Component({
  selector: 'employee-dashboard',
  template: `
    <employee-card>
      <label>Manager</label>
      <button (click)="fireSomeone($event)">Fire someone!</button>
    </employee-card>
  `
})
class EmployeeDashboard() {
  fireSomeone(event: Person) {}
}

```

```typescript
@Component({
  selector: 'employee-card',
  template: `
    <div class="card">
      <h3>Level</h3>
      <ng-content select="label"></ng-content>
      <h3>Take action</h3>
      <ng-content select="button"></ng-content>
  `
})
```

#### @ContentChild and ngAfterContentInit
- @ContentChild used to query projected content in the view
- ngAfterContentInit is hook to change data, subscribe to events. Happens before the view is initialized.
```typescript
import { ChildContent, AfterContentInit } from '@angular/core';

@Component({
  selector: 'employee-card',
  template: `
    <div class="card">
      <h3>Level</h3>
      <ng-content select="label"></ng-content>
      <h3>Take action</h3>
      <ng-content select="button"></ng-content>
  `
})
export class EmployeeCard implements AfterContentInit {
  @ContentChild(Button) button: Button;
  
  ngAfterContentInit() {
    if (this.button) {
      this.button.click.subscribe((click: ClickEvent) => console.log('clicked'));
    }
  }
}
```

#### @ContentChildren and QueryLists
```html
<employee-card>
  <button>Button 1</button>
  <button>Button 2</button>
</employee-card>
```

```typescript
import { ChildContent, AfterContentInit } from '@angular/core';

@Component({
  selector: 'employee-card',
  template: `
    <div class="card">
      <h3>Level</h3>
      <ng-content select="label"></ng-content>
      <h3>Take action</h3>
      <ng-content select="button"></ng-content>
  `
})
export class EmployeeCard implements AfterContentInit {
  @ContentChild(Button) buttons: QueryList<Button>;
  
  ngAfterContentInit() {
    if (this.buttons) {
      this.buttons.forEach(button => console.log(button.value))
    }
  }
}
```

#### @ViewChild and ngAfterViewInit
- different than @ContentChild (which queries a 'projected' element). @ViewChild will query the elements of the current component.
```typescript
import { 
  EmployeeCard,
  ViewChild,
  AfterViewInit, 
  AfterContentInit 
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div class="card">
      <EmployeeCard [employee]="employee"></EmployeeCard>
    </div>
  `
})
export class EmployeeDashboard implements AfterContentInit, AfterViewInit {
  employee: Employee
  @ViewChild(EmployeeCard) employeeCard: EmployeeCard;
  
  ngAfterViewInit() {
    // This will not work! This goes against Angular's uni-directional dataflow
    // We can subscribe to events and whatnot, just not change bindings
    this.employeeCard.employee = new Employee();
  }
  
  ngAfterContentInit() {
    // this will work
    if (this.employeeCard) {
      this.employeeCard.employee = new Employee();
    }
  }
}
```
#### @ViewChildren and QueryLists
- view children are not available in ngAferContentInit (unlike @ViewChild) because it is a dynamic 'live' list.
- if we need to mutate data after view initialized, we can use (hacky) the change detector:
```typescript
import { 
  ChangeDetectorRef,
  EmployeeCard,
  ViewChildren,
  QueryList,
  AfterViewInit, 
  AfterContentInit 
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div class="card">
      <EmployeeCard [employee]="employee"></EmployeeCard>
      <EmployeeCard [employee]="employee"></EmployeeCard>
      <EmployeeCard [employee]="employee"></EmployeeCard>
    </div>
  `
})
export class EmployeeDashboard implements AfterContentInit, AfterViewInit {
  employee: Employee
  @ViewChildren(EmployeeCard) employees: QueryList<EmployeeCard>
  
  constructor(private cd: ChangeDetectorRef) {}
  
  ngAfterViewInit() {
    if (this.employees) {
      this.employees.forEach(e => e.employee = new Employee());
      this.cd.detectCahnges();
    }
  }
  
  ngAfterContentInit() {
    // not available here
    this.employees;
  }
}
```
#### @ViewChild and template refs. ElementRef and nativeElement
```typescript
import { 
  ElementRef,
  AfterViewInit
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div>
      <input type="email" #email />
    </div>
  `
})
export class EmployeeDashboard implements AfterViewInit {
  
  @ViewChild('email') email: ElementRef;
 
  ngAfterViewInit() {
    console.log(email);
    console.log(email.nativeElement); // <input type="email"...
    // set the focus
    email.nativeElement.focus();
  }
}
```
#### Using the platform agnostic Renderer
- in Angular v5 and above, use __Renderer2__ instead. This will eventually be renamed
- platform agnostic. Can deploy to mobile and web application
```typescript
import { 
  ElementRef,
  Renderer,
  AfterViewInit
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div>
      <input type="email" #email />
    </div>
  `
})
export class EmployeeDashboard implements AfterViewInit {
  
  @ViewChild('email') email: ElementRef;
 
  constructor(private renderer: Renderer) {}
  
  ngAfterViewInit() {
    this.renderer.setElementAttribute(this.email.nativeElement, 'placeholder', 'Enter your email');
    this.renderer.invokeElementMethod(this.email.nativeElement, 'focus');
  }
}
```
#### Dynamic components with ComponentFactoryResolver
To inject a dynamic component:
1. setup an 'target' element using template refs
2. setup a `@ViewChild` as a `ViewContainerRef`
3. inject the `ComponenetFactoryResolver`
4. create componentFactory and call `createComponent` on view child to create the component as a sibling of 'target'
5. add to `entryComponents: []` in module

```typescript
import { 
  EmployeeCard,
  ComponentFactoryResolver,
  ViewContainerRef,
  ViewChild,
  AfterContentInit
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div #target></div>
  `
})
export class EmployeeDashboard implements AfterContentInit {
  
  @ViewChild('target', { read: ViewContainerRef }) target: ViewContainerRef;
 
  constructor(private resolver: ComponentFactoryResolver) {}
  
  ngAfterContentInit() {
    const componentFactory = this.resolver.resolveComponentFactory(EmployeeCard);
    const component = this.target.createComponent(componentFactory);
  }
```

```typescript
// feature.module.ts
import { NgModule } from '@angular/core';
import { EmployeeCard} from './employee-card.component';

@NgModule({
  entryComponents: [
    EmployeeCard
  ]
})
export class FeatureModule {}
```
- can access public properties on components without `@Input`

```typescript
  ngAfterContentInit() {
    ...
    const component = this.target.createComponent(componentFactory);
    component.instance.employee = new Employee();
  }

...

class EmployeeCard {
  // doe not need @Input()
  employee: Employee;
} 
```
- can subscribe to outputs

```typescript
  ngAfterContentInit() {
    ...
    const component = this.target.createComponent(componentFactory);
    component.instance.click.subscribe(this.clickHandler);
  }

...

class EmployeeCard {
  @Output() click: EventEmitter<Employee> = new EventEmitter<Employee>();
} 
```
#### Destroying dynamic components
```typescript
  ngAfterContentInit() {
    ...
    const component = this.target.createComponent(componentFactory);
    component.destroy();
  }
```

#### Moving / reordering dynamic components
```typescript
import { 
  EmployeeCardComponent,
  ComponentFactoryResolver,
  ComponentRef,
  ViewContainerRef,
  ViewChild,
  AfterContentInit
} from '@angular/core';

@Component({
  selector: 'employee-dashboard',
  template: `
    <div #target></div>
  `
})
export class EmployeeDashboard implements AfterContentInit {
  
  component1: ComponentRef<EmplolyeeCardComponent>;
  component2: ComponentRef<EmplolyeeCardComponent>;
  
  @ViewChild('target', { read: ViewContainerRef }) target: ViewContainerRef;
 
  constructor(private resolver: ComponentFactoryResolver) {}
  
  ngAfterContentInit() {
    const componentFactory = this.resolver.resolveComponentFactory(EmployeeCardComponent);
    this.component1 = this.target.createComponent(componentFactory, 0);
    this.component2 = this.target.createComponent(componentFactory, 1);
  }
  
  moveComponent() {
    // move below component2
    this.target.move(this.component1.hostView, 1);
  }
```

#### Dynamic template rendering, with context
- passing the context like this is how `*ngFor` works under the hood.
- content of `template` will be inserted as sibling to `#entry`
```typescript
import { 
  Component, 
  TemplateRef, 
  ViewContainerRef, 
  ViewChild, 
  AfterContentInit
} from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <div>
      <div #entry></div>
      <template #tmpl let-name let-location="location">
        {{ name }} : {{ location }}
      </template>
    </div>
  `
})
export class AppComponent implements AfterContentInit {
  @ViewChild('entry', { read: ViewContainerRef }) entry: ViewContainerRef;
  @ViewChild('tmpl') tmpl: TemplateRef<any>;

  ngAfterContentInit() {
    // pass in the context
    this.entry.createEmbeddedView(this.tmpl, {
      $implicit: 'Motto Todd', // will match 'name'
      location: 'UK, England'
    });
  }
}
```
#### Dynamic template rendering with ngTemplateOutlet, with context
- gives you declarative way to inject a template into the DOM instead of using the API calls like we did in the above example

```typescript
@Component({
  selector: 'app-root',
  template: `
    <div>
      <ng-container
        [ngTemplateOutlet]="tmpl"
        [ngTemplateOutletContext]="ctx">
      </ng-container>
      <template #tmpl let-name let-location="location">
        {{ name }} : {{ location }}
      </template>
    </div>
  `
})
export class AppComponent {
  ctx = {
    $implicit: 'Todd Motto',
    location: 'England, UK'
  };
}
```

#### View Encapsulation and Shadow DOM
- 3 types of view encapsulation
- Emulated: emulates the effects of Shadow DOM. Styles get referenced to the particular component. This is the default.
- Native: in newer browsers, will use the Shadow DOM, which essentially creates a DOM within a DOM
- None: no encapsulation. Effectively writes global styles into the DOM.

```typescript
@Component({
  selector: 'my-widget',
  encapsulation: ViewEncapsulation.Emulated | Native | None
})
```

#### OnPush Change Detection and Immutability
- angular is faster and more efficient when we use immutable objects
- at runtime, angular will create a change detector on the component
- angular is faster at using object reference comparisons than comparing properties of an object
- __use immutable data structures alongside onPush strategy__ to make your angular app much faster
- onPush is a good choice for stateless components

```typescript
// outside changes to user properties will not cause a re-render
// inside changes will, but this is not typical use case
// typically used with stateless components
@Component({
  selector: 'example-one',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ user.name }}</div>`
})
export class MyComponent {
  @Input() user: User;
  @Output() changeName: EventEmitter<string> = new EventEmitter<string>();
  
  localChange() {
    this.user.name = 'changed!';
  }
  
  changeName() {
    this.changeName.emit('changed'); // container should immutably change user and pass new ref down, causing a re-render
  }
}
  
```

## Directives