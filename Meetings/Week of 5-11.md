## Questions
- `ANY_VALUE`'s return within the group is defined to be "nondeterministic". However it is not random and from the perspective of the storage engine well defined (e.g. in MySQL the first row to be read will end up being the result of `ANY_VALUE`). So in the mtr tests, should we specify the specific value of the `ANY_VALUE` output?
	- Yes, this is okay
---
Decided follow a similar implementation as MySQL for the ANY_VALUE function