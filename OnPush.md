# Performance tuning Data Binding in Angular with OnPush, Immutables and Observables
 
Immutables and observables help the Angular change-tracking mechanism to identify components which have GUIs in need of updating. This enables an SPA to exclude unchanged components from change tracking which in turn drastically enhances performance.  
 
Angular’s already good performance is drastically improved when immutables and observables are introduced. Thanks to its optimisation capacity, corresponding to benchmarks, Angular promises breath-taking performance even in the case of over 50,000 data bindings. Whilst a development team can cohesively use the underlying ideas, it even benefits from a selective implementation in performance-critical components. 

The present article looks at this in detail, using an example to illustrate how and why Angular applications can be optimised by immutables and observables. The author lays out the [complete example here](https://github.com/manfredsteyer/angular2-performance).

## Immutables

The clue is in the name: immutables are data structures that cannot be modified. If objects described by them change, the entre immutable must be replaced with a new one. To discover changes, frameworks need therefore only check whether the whole immutable was modified rather than being updated on individual features. Or, to put it more technically: the framework only needs to check if the object reference has changed.
 
The delay method in the following example illustrates just that. Its task is to notice a 15-minute delay of the first flight of an array. So as to narrow the example’s scope, no libraries are used here which would constrain the immutables’ semantics. The example uses conventional JavaScript objects instead and refrains from changing them.  
 
Delay defines variables out the outset which refer to the whole array and the flight concerned, calling them ``oldFlights`` and ``oldFlight``. It also creates a date object called ``oldFlightDate`` which represents the date of the flight. 
 
Finally, it creates three objects which reflect the new conditions. The ``newFlightDate`` object refers to the modified date. This is turn takes over the ``newFlight`` object as well as the other values of ``oldFlight``. In addition, ``newFlights`` refers to a new array which is made up of the new flight and the rest of the unchanged flights.
 
To reach these flights delay uses the slice array method. Since slice returns an array with the chosen input, the EcmaScript 2015 spread operator made up of three points comes into play. It inserts the items of the array here delivered by slice directly into the subordinate array, in turn preventing an array in the array. This also illustrates that routines can take over unchanged subtrees from old immutables without modifying them.
 
It then stows the new array in the this.flights exit variable, thereby modfying its object reference. The debugging tasks at the end demonstrate that an application can now very easily discover changes in an array or in individual flights: one sole comparison is enough for that to happen.
 
```
delay() {
 
    const ONE_MINUTE = 1000 * 60;
 
    let oldFlights = this.flights;
    let oldFlight = oldFlights[0];
    let oldFlightDate = new Date(oldFlight.date);
 
    let newFlightDate = new Date(oldFlightDate.getTime() + ONE_MINUTE * 15);
    let newFlight =  {
            id: oldFlight.id,
            from: oldFlight.from,
            to: oldFlight.to,
            date: newFlightDate.toISOString()
    };
    let newFlights = [
        newFlight,
        ...oldFlights.slice(1)
    ];
 
    this.flights = newFlights;
 
    console.debug("Array: " + (oldFlights == newFlights)); // false
    console.debug("#0: " + (oldFlights[0] == newFlights[0])); // false
    console.debug("#1: " + (oldFlights[1] == newFlights[1])); // true
 
}
```

 
## Data binding in Angular  
 
One Angular application is a component tree. The following figure shows this using a very simple example: the FlightSearch component loads an array with flights, iterates it and passes on each flight for presentation to a card component.
 
![](http://i.imgur.com/w1zli15.png)

Since by definition only events may change the condition of components in Angular applications, the SPA framework updates the relevant property bindings after the events have been processed.
 

## Immutables and Angular
 
Angular traverses the entire component tree by default when updating property bindings. Thanks to immutables, Angular can nevertheless recognise the many subtrees that are unaffected by the changes and exclude these to enhance performance. The following figure illustrates this. It is based on the principle that the method described above (Listing 1) has updated the flights array as well as a flight therein. Thanks to immutables, Angular can check unconstrained whether the data passed on via the property bindings were changed.
 
As mentioned above, only one sole comparison per object is necessary. In the case analysed, Angular recognises that only one of the flights that have been passed on has changed and only takes into account the subtree affected by that change. It ignores all other subtrees. Since in this process the SPA framework need only follow one sole path through the tree in the best case scenario, it represents huge potential in larger component trees.

![](http://i.imgur.com/rgbM0GK.png) 

So that a component can benefit from this process, it configures the OnPush change detection strategy for its change detector via the component decorator:  
 
```
import { Flight } from '../entities/flight';
import { Input, Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
    selector: 'flight-card',
    templateUrl: './flight-card.component.html',
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class FlightCard {
    @Input() item: Flight;
}
```
 
This in turn means that Angular takes it that immutables are behind all incoming property bindings. If Angular concludes through a comparison of the object references in the property binding that the latest events carried out have not changed these objects, it excludes the whole subtree from further consideration.
 
 
# Experiments with OnPush

To research the functioning of OnPush, there is the possibility of temporarily replacing the this.flights = newFlights; line in the delay method with the following counterpart:

```
this.flights[0].date = newFlightDate.toISOString();
```

This modification changes the concerned flight object directly without changing the object reference behind the array or the one behind the flight. Therefore, when OnPush is employed, Angular does not recognise the modification and the GUI remains unchanged. When OnPush is not used, Angular also recognises the direct change, since in that case it traverses the whole component tree.

 
 
# Observables and Angular

In addition to immutables, Angular also supports observables for performance optimisation. These objects can tell Angular if a new version of a related object exists. This in turn enables components to be excluded from the tracking of modifications until they receive such a notification.
 
The following figure illustrates this process by representing individual flights through observables. The use of the dollar sign ($) has been introduced as a suffix to highlight this fact. Both card components continue to use the OnPush strategy. Since the reference to the transferred observables does not change, Angular excludes it from the change tracking process. In any case, when receiving a new version of the flight the application can indicate to Angular that the data binding needs to be updated. In this instance, just as when immutables are used, Angular also checks all subordinate components for change, especially verifying that the data by definition flows from the top to the bottom, and thereby also that the data can be applied further above.  
 
![](http://i.imgur.com/wsjCpNn.png)
 
The following figure illustrates a similar thought experiment in this regard. It represents the array with the flights through an observable. If the ``FlightSearch`` component additionally uses the OnPush strategy, Angular firstly also excludes them from the change tracking process. If the observable flags up a new version of the array, Angular balances out the ``FlightSearch`` status with the GUI. The same is also true here of all superordinate components which are not visible on this figure. What happens in the case of subordinate card components depends on their change detection strategy. If immutables with OnPush are used here, as described above, Angular only takes into account the subtrees with changes. Otherwise it iterates all subordinate subtrees. 

![](http://i.imgur.com/CcBKHvZ.png)

 
# Data binding with observables
 
To illustrate the last process described above, the example explained below administers the requested flights using a ``ReplaySubject``:
 
```
import { ReplaySubject } from 'rxjs/ReplaySubject'
[...]
public flights$ = new ReplaySubject<Flight[]>(1);
```

 
In the example given above, the observable directly receives data to be sent via methods and forwards them to one or more Subscriber. Furthermore, the ``ReplaySubject`` caches the last sent objects so that a new Subscriber can receive them immediately. The size of the cache can be configured via the constructor. The example sends new versions of the flight array using the next method: 
 
```
this.flights$.next(newFlights);
```

Using the ``async`` pipe the observable can be used in data-binding expressions. For that it behaves like a Subscriber and takes care of GUI updates when new data arrives: 
 
``` 
<div *ngFor="let flight of flights$ | async">
    <flight-card [item]="flight"></flight-card>
</div>   
```

# Conclusion and outlook 
Thanks to immutables and observables Angular can be easily made aware of changed values without the need to traverse the entire component tree. As a result, in the best-case scenario only one path from the root to one of the affected nodes can be noted, and Angular need only check log(n) components rather than n components. For example, for 50,000 components this means that only a low one-figure amount of components needs to be taken into account. 
 
To benefit from these optimisations, there is no need for an application to be based exclusively on this data structure. The development team can, rather, go with a change detection strategy per component, thereby focusing on performance-critical sections. Using immutables does take some getting used to, especially since they are not directly supported by JavaScript. Amongst others, the programmer techniques presented here offer remedies, as do libraries such as [Immutable.js](https://facebook.github.io/immutable-js/), [seamless-immutable](https://github.com/rtfeldman/seamless-immutable) and [immutability-helper](https://www.npmjs.com/package/immutability-helper).

Anyone looking to exclude situations with optimisation needs from the get go by continuously using immutables and observables should opt for Redux implementation. Redux puts the named data structures centre stage and offers a structure for their use, thus leading not only to better performance, but also to improved sustainability. An example of such implementation is the [@ngrx/store](https://github.com/ngrx/store) library described here which originates from one of the members of the Angular team.