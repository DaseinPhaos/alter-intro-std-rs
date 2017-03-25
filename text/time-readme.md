% std::time

This module introduces temporal quantification into the language.

Mainly, two types are defined:

Type `Duration` represents a span of time:

```ignore
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Debug, Hash, Default)]
pub struct Duration {
    secs: u64,
    nanos: u32,
}
```

To construct it:

```ignore
impl Duration {
    pub fn new(secs: u64, nanos: u32) -> Duration {
        let secs = secs.checked_add(nanos / NANOS_PER_SEC) as u64)
            .expect("overflow in Duration::new");
        let nanos = nanos % NANOS_PER_SEC;
        Duration { secs: secs, nanos: nanos }
    }

    pub fn from_secs(secs: u64) -> Duration {
        Duration { secs: secs, nanos: 0 }
    }

    pub fn from_millis(millis: u64) -> Duration {
        let secs = millis / MILLIS_PER_SEC;
        let nanos = ((millis % MILLIS_PER_SEC) as u32) * NANOS_PER_MILLI;
        Duration { secs: secs, nanos: nanos }
    }
}
```

To access its fields:

```ignore
impl Duration {
    pub fn as_secs(&self) -> u64 {
        self.secs
    }

    pub fn subsec_nanos(&self) -> u32 {
        self.nanos
    }
}
```

To perform checked arithmetic operations:

```
impl Duration {
    pub fn checked_add(self, rhs: Duration) -> Option<Duration> {
        if let Some(mut secs) = self.secs.checked_add(rhs.secs) {
            let mut nanos = self.nanos + rhs.nanos;
            if nanos >= NANOS_PER_SEC {
                nanos -= NANOS_PER_SEC;
                if let Some(new_secs) = secs.checked_add(1) {
                    secs = new_secs;
                } else {
                    return None;
                }
            }
            debug_assert!(nanos < NANOS_PER_SEC);
            Some(Duration {
                secs: secs,
                nanos: nanos,
            })
        } else {
            None
        }
    }

    pub fn checked_sub(self, rhs: Duration) -> Option<Duration> {
        if let Some(mut secs) = self.secs.checked_sub(rhs.secs) {
            let mut nanos = if self.nanos >= rhs.nanos {
                self.nanos - rhs.nanos
            } else {
                if let Some(sub_secs) = secs.checked_sub(1) {
                    secs = sub_secs;
                    self.nanos + NANOS_PER_SEC - rhs.nanos
                } else {
                    return None;
                }
            };
            debug_assert!(nanos < NANOS_PER_SEC); // This assertion also guards against subtraction overflow 
            Some(Duration { secs: secs, nanos: nanos })
        } else {
            None
        }
    }

    pub fn checked_mul(self, rhs: u32) -> Option<Duration> {
        let total_nanos = self.nanos as u64 * rhs as u64;
        let extra_secs = total_nanos / (NANOS_PER_SEC as u64);
        let nanos = (total_nanos % (NANOS_PER_SEC as u64)) as u32;
        if let Some(secs) = self.secs
            .checked_mul(rhs as u64)
            .and_then(|s| s.checked_add(extra_secs)) {
            debug_assert!(nanos < NANOS_PER_SEC);
            Some(Duration {
                secs: secs,
                nanos: nanos,
            })
        } else {
            None
        }
    }

    pub fn checked_div(self, rhs: u32) -> Option<Duration> {
        if rhs != 0 {
            let secs = self.secs / (rhs as u64);
            let carry = self.secs - secs * (rhs as u64);
            let extra_nanos = carry * (NANOS_PER_SEC as u64) / (rhs as u64);
            let nanos = self.nanos / rhs + (extra_nanos as u32);
            debug_assert!(nanos < NANOS_PER_SEC);
            Some(Duration { secs: secs, nanos: nanos })
        } else {
            None
        }
    }
}
```

Also note that `Add`, `AddAssign`, `Sub`, `SubAssign`, `Mul<u32>`, `MulAssign<u32>`, `Div<u32>`, `DivAssign<u32>` are also implemented for `Duration`. The implementations just wrap around the `checked` methods respectively.

Type `Instant` struct, on the other hand, represents a time point.

To obtain a new instant, we call

```ignore
impl Instant {
    pub fn now() -> Instant
}
```

This method returns a time point corresponding to "now". A sequence of invocations of this method are guaranteed to return a sequence of monotonically increasing instants. Thus, we say that `Instant` provides a measurement of an monotonically increasing clock.

To obtain the duration between two instants:

```ignore
impl Instant {
    pub fn duration_since(&self, earlier: Instant) -> Duration
}
```

This method would panic if `earlier` is an instant that is obtained later than `self`.

To obtain the duration from the instant until "now":

```ignore
impl Instant {
    pub fn elapsed(&self) -> Duration
}
```

This method would panic if `self` represents an instant later than "now".

Also note that, `Add<Duration, Output = Instant>`, `AddAssign<Duration>`, `Sub<Duration, Output = Instant>`, `SubAssign<Duration>`, and `Sub<Instant, Output = Duration` are implemented for `Instant`.
