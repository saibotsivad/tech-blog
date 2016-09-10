---
title: Some random notes about Java Strings
author: Tobias Davis
layout: post
---

During a recent big project we hired about a dozen contractors to do a lot of the initial "heavy-lifting"
work for us. While this has worked well, there have been a number of Java-based string manipulation issues
that do not seem to be obvious, so I'm writing them down here.

# Using `string.replaceAll` is slow

If you read the JavaDocs you'll see that `String` has the method `String.replaceAll` which accepts regex, e.g.:

	public class MyClass {
		public String cleaner(input) {
			return input.replaceAll("[0-9]+", "X");
		}
		public void test() {
			Assert.assertEquals("aXbX", cleaner("a1b2"));
		}
	}

However, this approach means that the regex is compiled to a `Pattern` *each time* this is run. This is
a slow process, and it can often be changed to this instead:

	public class MyClass {
		private static final Pattern NUMBER_PATTERN = Pattern.compile("[0-9]+");
		public String cleaner(input) {
			return NUMBER_PATTERN.matcher(input).replaceAll("X");
		}
		public void test() {
			Assert.assertEquals("aXbX", cleaner("a1b2"));
		}
	}

---

# Using `string.matches` is slow

For the same reason that the `string.replaceAll` is slow, the `string.matches` is also slow (the `Pattern` is compiled each time
the method is called). So this approach is slow:

	public class MyClass {
		public String doesItMatch(input) {
			return input.matches("^[0-9]+$");
		}
		public void test() {
			Assert.assertTrue(doesItMatch("123"));
		}
	}

And this approach is faster:

	public class MyClass {
		private static final Pattern NUMBER_PATTERN = Pattern.compile("[0-9]+");
		public String doesItMatch(input) {
			return NUMBER_PATTERN.matcher(input).matches();
		}
		public void test() {
			Assert.assertTrue(doesItMatch("123"));
		}
	}

---
