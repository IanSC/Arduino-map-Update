# Arduino map() Update
Fix map() function when using high value parameters.  
Run map_problem.ino

## PROBLEM

Arduino's map() function accepts long and returns long, however it gives wrong results
even if valid but high value parameters are used.

```
long map(long x, long in_min, long in_max, long out_min, long out_max) { ... }

map( 9000000, 0, 10000000, 0, 1000 );
   should be: 9m/10m = 0.9 x 1000 ==> 900
   but map() returns: 41
        
map( 750, 0, 1000, 0, 10000000 );
   shold be: 750/1000 = 0.75 x 10,000,000 ==> 7,500,000
   but map() returns: -1089934
```
This happens because of overflow. Several alternatives were tested for speed and accuracy.
The ones with less impact for non-overflow conditions were prioritized.

To check for multiplication overflow:  
https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html  
https://stackoverflow.com/a/57260786/351245

## RESULTS

Conclusion:
- use "long long", "float" introduces rounding discrepancies
- for ESP32 and Seeduino Xiao, check first if there is overflow before fixing
- for Arduino Uno, skip checking
- don't bother saving interim results, either compiler optimizes it anyway or it is just slower
```
WITH CHECKING, if overflow run alternate code:
   Arduino:  +40 % if there is no overflow
             +98 % if there is overflow to fix
   ESP32:     +6 % if there is no overflow
            +251 % if there is overflow to fix
   Xiao:     +90 % if there is no overflow
            +490 % if there is overflow to fix

long mapCheckAndFloatFix2(long x, long in_min, long in_max, long out_min, long out_max ) {
   long result;
   if ( __builtin_smull_overflow( x - in_min, out_max - out_min, &result ) ) {
      return ( (float) (x - in_min) * (float) (out_max - out_min) ) / (float) (in_max - in_min) + out_min;
   }
   return result / (in_max - in_min) + out_min;
}

NO CHECKING, always run overflow safe code:
   Arduino:  +33 % if there is no overflow
             +56 % if there is overflow to fix
   ESP32:   +269 % if there is no overflow
            +271 % if there is overflow to fix
   Xiao:    +394 % if there is no overflow
            +442 % if there is overflow to fix

long mapLongLong2( long x, long in_min, long in_max, long out_min, long out_max ) {
   return ( (long long)(x - in_min) * (long long)(out_max - out_min) ) / (in_max - in_min) + out_min;
}
```
## BENCHMARK
```
================================
 Arduino Uno
 - each run 11000 iterations
 - averaged result from 10 runs
================================

run 1: map (native library)    = 518.90
run 1: mapArduino              = 518.90
run 1: mapESP32                = 541.00
run 1: mapCheckNoFix           = 516.20
run 1: mapCheckAndFloatFix1    = 894.90   *
run 1: mapCheckAndFloatFix2    = 941.60   *
run 1: mapCheckAndLongLongFix1 = 1030.60
run 1: mapCheckAndLongLongFix2 = 1030.90  <-- cost to fix = 512 ms for 11000x, +98 %
run 1: mapFloat1               = 691.80   *
run 1: mapFloat2               = 681.80   *
run 1: mapLongLong1            = 803.50
run 1: mapLongLong2            = 803.30   <-- no checking is faster !!! cost = 293.4 ms, +56 %

run 2: map (native library)    = 517.60
run 2: mapArduino              = 517.30
run 2: mapESP32                = 539.40
run 2: mapCheckNoFix           = 514.90
run 2: mapCheckAndFloatFix1    = 727.50  *
run 2: mapCheckAndFloatFix2    = 774.20  *
run 2: mapCheckAndLongLongFix1 = 725.90
run 2: mapCheckAndLongLongFix2 = 725.90  <-- nothing to fix, cost = 208.6 ms for 11000x, +40 %
run 2: mapFloat1               = 717.30  *
run 2: mapFloat2               = 707.60  *
run 2: mapLongLong1            = 688.10
run 2: mapLongLong2            = 688.10  <-- no checking is faster !!! cost = 170.8 ms, +33 %

map (native library)    = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934, 
mapArduino              = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934, 
mapESP32                = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934, 
mapCheckNoFix           = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934, 
mapCheckAndFloatFix1    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapCheckAndFloatFix2    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapCheckAndLongLongFix1 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapCheckAndLongLongFix2 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapFloat1               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapFloat2               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapLongLong1            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapLongLong2            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 

map (native library)    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapArduino              = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- AUTHORITATIVE
mapESP32                = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckNoFix           = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix1    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix2    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix1 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix2 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapFloat1               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapFloat2               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapLongLong1            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapLongLong2            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 

=================================
 ESP32
 - each run 11000 iterations
 - averaged result from 100 runs
=================================

run 1: map (native library)    = 1.44  <-- ESP32 orig
run 1: mapArduino              = 1.20  <-- Arduino's version
run 1: mapESP32                = 1.36  <-- ESP32 with my pull request, no fix yet
run 1: mapCheckNoFix           = 1.18
run 1: mapCheckAndFloatFix1    = 9.03  *
run 1: mapCheckAndFloatFix2    = 8.41  *
run 1: mapCheckAndLongLongFix1 = 5.40
run 1: mapCheckAndLongLongFix2 = 4.77  <-- cost to fix = 3.41 ms for 10000x, +251 %
run 1: mapFloat1               = 9.00  *
run 1: mapFloat2               = 9.00  *
run 1: mapLongLong1            = 5.07
run 1: mapLongLong2            = 5.05  <-- no checking is comparable, cost = 3.69 ms, +271 %

run 2: map (native library)    = 1.41  <-- ESP32 orig
run 2: mapArduino              = 1.17
run 2: mapESP32                = 1.32  <-- ESP32 with my pull request, no fix yet
run 2: mapCheckNoFix           = 1.17
run 2: mapCheckAndFloatFix1    = 2.03  *
run 2: mapCheckAndFloatFix2    = 1.41  *
run 2: mapCheckAndLongLongFix1 = 2.05
run 2: mapCheckAndLongLongFix2 = 1.40  <-- nothing to fix, cost = 0.08 ms for 11000x, +6 %
run 2: mapFloat1               = 8.99  *
run 2: mapFloat2               = 8.99  *
run 2: mapLongLong1            = 4.87  
run 2: mapLongLong2            = 4.88  <-- if checking is skipped, cost = 3.56 ms, +269 %

map (native library)    = 41, 901, -200, -99, 10000201, 9999873, 71, -108, 705033, -1089934,         <-- WRONG
mapArduino              = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapESP32                = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapCheckNoFix           = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapCheckAndFloatFix1    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapCheckAndFloatFix2    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapCheckAndLongLongFix1 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapCheckAndLongLongFix2 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapFloat1               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapFloat2               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapLongLong1            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapLongLong2            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 

map (native library)    = 98, 99, 28, 696, 92, 2501, 5001, 7500, 5000, 80,  <-- INCONSISTENT
mapArduino              = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- AUTHORITATIVE
mapESP32                = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- my pull request to ESP32, no overflow fix
mapCheckNoFix           = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix1    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix2    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix1 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix2 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapFloat1               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapFloat2               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapLongLong1            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapLongLong2            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 

================================
Seeeduino Xiao
- each run 11000 iterations
- averaged result from 10 runs
================================

run 1: map (native library)    = 31.70
run 1: mapArduino              = 31.60
run 1: mapESP32                = 32.40
run 1: mapCheckNoFix           = 31.50
run 1: mapCheckAndFloatFix1    = 247.90
run 1: mapCheckAndFloatFix2    = 252.60
run 1: mapCheckAndLongLongFix1 = 182.80
run 1: mapCheckAndLongLongFix2 = 187.20  <-- cost to fix = 155.5 ms for 11000x, +490 %
run 1: mapFloat1               = 250.70
run 1: mapFloat2               = 248.30
run 1: mapLongLong1            = 172.00
run 1: mapLongLong2            = 172.00  <-- if checking is skipped, cost = 140.3 ms, +442 %

run 2: map (native library)    = 32.00
run 2: mapArduino              = 32.00
run 2: mapESP32                = 32.20
run 2: mapCheckNoFix           = 32.00
run 2: mapCheckAndFloatFix1    = 55.70
run 2: mapCheckAndFloatFix2    = 60.00
run 2: mapCheckAndLongLongFix1 = 56.30
run 2: mapCheckAndLongLongFix2 = 60.90   <-- nothing to fix, cost = 28.90 ms for 11000x, +90 %
run 2: mapFloat1               = 255.80
run 2: mapFloat2               = 253.00
run 2: mapLongLong1            = 158.00
run 2: mapLongLong2            = 158.20  <-- if checking is skipped, cost = 126.2 ms, +394 %

map (native library)    = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapArduino              = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapESP32                = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapCheckNoFix           = 41, 900, -200, -100, 10000200, 9999872, 70, -108, 705032, -1089934,        <-- WRONG
mapCheckAndFloatFix1    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT 
mapCheckAndFloatFix2    = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapCheckAndLongLongFix1 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapCheckAndLongLongFix2 = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapFloat1               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapFloat2               = 899, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000,  <-- INCONSISTENT
mapLongLong1            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 
mapLongLong2            = 900, 900, 5000000, 2500000, 5000000, 2500000, 500, 750, 5000000, 7500000, 

map (native library)    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapArduino              = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- AUTHORITATIVE
mapESP32                = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckNoFix           = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix1    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndFloatFix2    = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix1 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapCheckAndLongLongFix2 = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapFloat1               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapFloat2               = 97, 97, 28, 695, 91, 2500, 5000, 7500, 5000, 80,  <-- INCONSISTENT
mapLongLong1            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
mapLongLong2            = 97, 98, 28, 695, 91, 2500, 5000, 7500, 5000, 80, 
```
## CANDIDATES
```
long mapArduino(long x, long in_min, long in_max, long out_min, long out_max) {
    return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
}

long mapESP32( long value, long inputMin, long inputMax, long outputMin, long outputMax ) {
    // not original
    const long run = inputMax - inputMin;
    if( run == 0 ) {
        #if defined(ESP32)
        log_e( "spMap(): invalid input range, min == max" );
        #endif
        return -1; //AVR returns -1, SAM returns 0
    }
    const long rise = outputMax - outputMin;
    const long delta = value - inputMin;
    // return ( delta * dividend + (divisor / 2) ) / divisor + outputMin;
    return ( delta * rise ) / run + outputMin;
}

long mapCheckNoFix(long x, long in_min, long in_max, long out_min, long out_max) {
    // with detection but not fixed, determine cost of checking    
    long result;
    if ( __builtin_smull_overflow( (x - in_min), (out_max - out_min), &result) ) {
    }
    return result / (in_max - in_min) + out_min;
}

long mapCheckAndFloatFix1(long x, long in_min, long in_max, long out_min, long out_max) {
    // with detection and fix, save interim results and reuse
    long result;
    long delta = x - in_min;
    long rise  = out_max - out_min;
    if ( __builtin_smull_overflow( delta, rise, &result ) ) {
        return ( (float) delta * (float) rise ) / (float) (in_max - in_min) + out_min;
    }
    return result / (in_max - in_min) + out_min;
}

long mapCheckAndFloatFix2(long x, long in_min, long in_max, long out_min, long out_max ) {
    // with detection and fix. do not save interim results
    long result;
    if ( __builtin_smull_overflow( x - in_min, out_max - out_min, &result ) ) {
        return ( (float) (x - in_min) * (float) (out_max - out_min) ) / (float) (in_max - in_min) + out_min;
    }
    return result / (in_max - in_min) + out_min;
}

long mapCheckAndLongLongFix1(long x, long in_min, long in_max, long out_min, long out_max) {
    // with detection and fixed, save interim results and reuse
    long result;
    long delta = x - in_min;
    long rise  = out_max - out_min;
    if ( __builtin_smull_overflow( delta, rise, &result ) ) {
        return ( (long long) delta * (long long) rise ) / ( in_max - in_min ) + out_min;
    }
    return result / (in_max - in_min) + out_min;
}

long mapCheckAndLongLongFix2(long x, long in_min, long in_max, long out_min, long out_max) {
    // with detection and fixed, do not save interim results
    long result;
    if ( __builtin_smull_overflow( x - in_min, out_max - out_min, &result ) ) {
        return ( (long long) ( x - in_min ) * (long long) ( out_max - out_min ) ) / ( in_max - in_min ) + out_min;
    }
    return result / (in_max - in_min) + out_min;
}

long mapFloat1(long x, long in_min, long in_max, long out_min, long out_max) {
    // sometimes INCORRECT, depending on rounding
    float out   = out_max - out_min;
    float in    = in_max - in_min;
    float delta = x - in_min;
    return ( delta * out ) / in + out_min;   
    // return round( ( delta * out ) / in + out_min );
}

long mapFloat2(long x, long in_min, long in_max, long out_min, long out_max) {
    // sometimes INCORRECT, depending on rounding
    return ( (float) (x - in_min) * (float) (out_max - out_min) ) / (float) (in_max - in_min) + out_min;
    //return round( ( (float) (x - in_min) * (float) (out_max - out_min) ) / (float) (in_max - in_min) ) + out_min;
}

long mapLongLong1( long x, long in_min, long in_max, long out_min, long out_max ) {
    // correct answers
    long long delta = x - in_min;
    long long rise  = out_max - out_min;
    long      run   = in_max - in_min;
    return ( delta * rise ) / run + out_min;
}

long mapLongLong2( long x, long in_min, long in_max, long out_min, long out_max ) {
    return ( (long long)(x - in_min) * (long long)(out_max - out_min) ) / (in_max - in_min) + out_min;
}
```
