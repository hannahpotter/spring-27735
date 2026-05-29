---
isPR: true
prNumber: 1
prOwner: hannahpotter
prRepo: spring-27735
baseRef: prestate
---
# DataAccessUtils result accessors with Optional return value

This PR extends `DataAccessUtils` with new `optionalResult` methods and overloads for `singleResult` that accept `Stream` and `Iterator`. These additions provide more idiomatic ways to access single results using Java 8 structures like `Optional` and `Stream`.

## Core API Updates in DataAccessUtils

The `DataAccessUtils` class now includes imports for standard Java collections and stream components. Most importantly, it introduces new extraction methods that can handle `Stream` and `Iterator` structures gracefully, and return `Optional` values to represent nullable singular results.

:::hunk file=spring-tx/src/main/java/org/springframework/dao/support/DataAccessUtils.java level=2
@@ -17,6 +17,11 @@
 package org.springframework.dao.support;
 
 import java.util.Collection;
+import java.util.Iterator;
+import java.util.List;
+import java.util.Optional;
+import java.util.stream.Stream;
+import java.util.stream.StreamSupport;
 
 import org.springframework.dao.DataAccessException;
 import org.springframework.dao.EmptyResultDataAccessException;
:::

:::hunk file=spring-tx/src/main/java/org/springframework/dao/support/DataAccessUtils.java level=2
@@ -56,6 +61,109 @@ public static <T> T singleResult(@Nullable Collection<T> results) throws Incorre
 		return results.iterator().next();
 	}
 
+	/**
+	 * Return a single result object from the given Stream.
+	 * <p>Returns {@code null} if 0 result objects found;
+	 * throws an exception if more than 1 element found.
+	 * @param results the result Stream (can be {@code null})
+	 * @return the single result object, or {@code null} if none
+	 * @throws IncorrectResultSizeDataAccessException if more than one
+	 * element has been found in the given Stream
+	 */
+	@Nullable
+	public static <T> T singleResult(@Nullable Stream<T> results) throws IncorrectResultSizeDataAccessException {
+		if (results == null) {
+			return null;
+		}
+		List<T> resultList = results.toList();
+		if (resultList.size() > 1) {
+			throw new IncorrectResultSizeDataAccessException(1, resultList.size());
+		}
+		return resultList.stream().findFirst().orElse(null);
+	}
+
+	/**
+	 * Return a single result object from the given Iterator.
+	 * <p>Returns {@code null} if 0 result objects found;
+	 * throws an exception if more than 1 element found.
+	 * @param results the result Iterator (can be {@code null})
+	 * @return the single result object, or {@code null} if none
+	 * @throws IncorrectResultSizeDataAccessException if more than one
+	 * element has been found in the given Iterator
+	 */
+	@Nullable
+	public static <T> T singleResult(@Nullable Iterator<T> results) throws IncorrectResultSizeDataAccessException {
+		if (results == null) {
+			return null;
+		}
+		Iterable<T> iterable = () -> results;
+		List<T> resultList = StreamSupport.stream(iterable.spliterator(), false).toList();
+		if (resultList.size() > 1) {
+			throw new IncorrectResultSizeDataAccessException(1, resultList.size());
+		}
+		return resultList.stream().findFirst().orElse(null);
+	}
+
+	/**
+	 * Return a single result object from the given Collection.
+	 * <p>Returns {@code Optional.empty()} if 0 result objects found;
+	 * throws an exception if more than 1 element found.
+	 * @param results the result Collection (can be {@code null})
+	 * @return the single optional result object, or {@code Optional.empty()} if none
+	 * @throws IncorrectResultSizeDataAccessException if more than one
+	 * element has been found in the given Collection
+	 */
+	public static <T> Optional<T> optionalResult(@Nullable Collection<T> results) throws IncorrectResultSizeDataAccessException {
+		if (CollectionUtils.isEmpty(results)) {
+			return Optional.empty();
+		}
+		if (results.size() > 1) {
+			throw new IncorrectResultSizeDataAccessException(1, results.size());
+		}
+		return results.stream().findFirst();
+	}
+
+	/**
+	 * Return a single result object from the given Stream.
+	 * <p>Returns {@code Optional.empty()} if 0 result objects found;
+	 * throws an exception if more than 1 element found.
+	 * @param results the result Stream (can be {@code null})
+	 * @return the single optional result object, or {@code Optional.empty()} if none
+	 * @throws IncorrectResultSizeDataAccessException if more than one
+	 * element has been found in the given Stream
+	 */
+	public static <T> Optional<T> optionalResult(@Nullable Stream<T> results) throws IncorrectResultSizeDataAccessException {
+		if (results == null) {
+			return Optional.empty();
+		}
+		List<T> resultList = results.toList();
+		if (resultList.size() > 1) {
+			throw new IncorrectResultSizeDataAccessException(1, resultList.size());
+		}
+		return resultList.stream().findFirst();
+	}
+
+	/**
+	 * Return a single result object from the given Iterator.
+	 * <p>Returns {@code Optional.empty()} if 0 result objects found;
+	 * throws an exception if more than 1 element found.
+	 * @param results the result Iterator (can be {@code null})
+	 * @return the single optional result object, or {@code Optional.empty()} if none
+	 * @throws IncorrectResultSizeDataAccessException if more than one
+	 * element has been found in the given Iterator
+	 */
+	public static <T> Optional<T> optionalResult(@Nullable Iterator<T> results) throws IncorrectResultSizeDataAccessException {
+		if (results == null) {
+			return Optional.empty();
+		}
+		Iterable<T> iterable = () -> results;
+		List<T> resultList = StreamSupport.stream(iterable.spliterator(), false).toList();
+		if (resultList.size() > 1) {
+			throw new IncorrectResultSizeDataAccessException(1, resultList.size());
+		}
+		return resultList.stream().findFirst();
+	}
+
 	/**
 	 * Return a single result object from the given Collection.
 	 * <p>Throws an exception if 0 or more than 1 element found.
:::

## Testing the New Accessors

We've updated `DataAccessUtilsTests` to exercise the newly added API. This includes validating empty collections, large collections (to ensure they throw typical exceptions), and correctly handling numerous primitive wrapper combinations.

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -47,6 +41,13 @@ public void withEmptyCollection() {
 
 		assertThat(DataAccessUtils.uniqueResult(col)).isNull();
 
+		assertThat(DataAccessUtils.singleResult(col)).isNull();
+		assertThat(DataAccessUtils.singleResult(col.stream())).isNull();
+		assertThat(DataAccessUtils.singleResult(col.iterator())).isNull();
+		assertThat(DataAccessUtils.optionalResult(col)).isEmpty();
+		assertThat(DataAccessUtils.optionalResult(col.stream())).isEmpty();
+		assertThat(DataAccessUtils.optionalResult(col.iterator())).isEmpty();
+
 		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
 				DataAccessUtils.requiredUniqueResult(col))
 			.satisfies(sizeRequirements(1, 0));
:::

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -89,6 +90,30 @@ public void withTooLargeCollection() {
 		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
 				DataAccessUtils.longResult(col))
 			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.singleResult(col))
+			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.singleResult(col.stream()))
+			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.singleResult(col.iterator()))
+			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.optionalResult(col))
+			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.optionalResult(col.stream()))
+			.satisfies(sizeRequirements(1, 2));
+
+		assertThatExceptionOfType(IncorrectResultSizeDataAccessException.class).isThrownBy(() ->
+				DataAccessUtils.optionalResult(col.iterator()))
+			.satisfies(sizeRequirements(1, 2));
 	}
 
 	@Test
:::

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -102,6 +127,12 @@ public void withInteger() {
 		assertThat(DataAccessUtils.objectResult(col, String.class)).isEqualTo("5");
 		assertThat(DataAccessUtils.intResult(col)).isEqualTo(5);
 		assertThat(DataAccessUtils.longResult(col)).isEqualTo(5);
+		assertThat(DataAccessUtils.singleResult(col)).isEqualTo(5);
+		assertThat(DataAccessUtils.singleResult(col.stream())).isEqualTo(5);
+		assertThat(DataAccessUtils.singleResult(col.iterator())).isEqualTo(5);
+		assertThat(DataAccessUtils.optionalResult(col)).isEqualTo(Optional.of(5));
+		assertThat(DataAccessUtils.optionalResult(col.stream())).isEqualTo(Optional.of(5));
+		assertThat(DataAccessUtils.optionalResult(col.iterator())).isEqualTo(Optional.of(5));
 	}
 
 	@Test
:::

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -139,6 +170,12 @@ public void withLong() {
 		assertThat(DataAccessUtils.objectResult(col, String.class)).isEqualTo("5");
 		assertThat(DataAccessUtils.intResult(col)).isEqualTo(5);
 		assertThat(DataAccessUtils.longResult(col)).isEqualTo(5);
+		assertThat(DataAccessUtils.singleResult(col)).isEqualTo(Long.valueOf(5L));
+		assertThat(DataAccessUtils.singleResult(col.stream())).isEqualTo(Long.valueOf(5L));
+		assertThat(DataAccessUtils.singleResult(col.iterator())).isEqualTo(Long.valueOf(5L));
+		assertThat(DataAccessUtils.optionalResult(col)).isEqualTo(Optional.of(5L));
+		assertThat(DataAccessUtils.optionalResult(col.stream())).isEqualTo(Optional.of(5L));
+		assertThat(DataAccessUtils.optionalResult(col.iterator())).isEqualTo(Optional.of(5L));
 	}
 
 	@Test
:::

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -149,6 +186,12 @@ public void withString() {
 		assertThat(DataAccessUtils.uniqueResult(col)).isEqualTo("test1");
 		assertThat(DataAccessUtils.requiredUniqueResult(col)).isEqualTo("test1");
 		assertThat(DataAccessUtils.objectResult(col, String.class)).isEqualTo("test1");
+		assertThat(DataAccessUtils.singleResult(col)).isEqualTo("test1");
+		assertThat(DataAccessUtils.singleResult(col.stream())).isEqualTo("test1");
+		assertThat(DataAccessUtils.singleResult(col.iterator())).isEqualTo("test1");
+		assertThat(DataAccessUtils.optionalResult(col)).isEqualTo(Optional.of("test1"));
+		assertThat(DataAccessUtils.optionalResult(col.stream())).isEqualTo(Optional.of("test1"));
+		assertThat(DataAccessUtils.optionalResult(col.iterator())).isEqualTo(Optional.of("test1"));
 
 		assertThatExceptionOfType(TypeMismatchDataAccessException.class).isThrownBy(() ->
 				DataAccessUtils.intResult(col));
:::

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -167,6 +210,12 @@ public void withDate() {
 		assertThat(DataAccessUtils.requiredUniqueResult(col)).isEqualTo(date);
 		assertThat(DataAccessUtils.objectResult(col, Date.class)).isEqualTo(date);
 		assertThat(DataAccessUtils.objectResult(col, String.class)).isEqualTo(date.toString());
+		assertThat(DataAccessUtils.singleResult(col)).isEqualTo(date);
+		assertThat(DataAccessUtils.singleResult(col.stream())).isEqualTo(date);
+		assertThat(DataAccessUtils.singleResult(col.iterator())).isEqualTo(date);
+		assertThat(DataAccessUtils.optionalResult(col)).isEqualTo(Optional.of(date));
+		assertThat(DataAccessUtils.optionalResult(col.stream())).isEqualTo(Optional.of(date));
+		assertThat(DataAccessUtils.optionalResult(col.iterator())).isEqualTo(Optional.of(date));
 
 		assertThatExceptionOfType(TypeMismatchDataAccessException.class).isThrownBy(() ->
 				DataAccessUtils.intResult(col));
:::

## Miscellaneous

This section groups minor or mechanical changes, such as organizing test imports.

:::hunk file=spring-tx/src/test/java/org/springframework/dao/support/DataAccessUtilsTests.java level=2
@@ -16,13 +16,7 @@
 
 package org.springframework.dao.support;
 
-import java.util.ArrayList;
-import java.util.Arrays;
-import java.util.Collection;
-import java.util.Date;
-import java.util.HashMap;
-import java.util.HashSet;
-import java.util.Map;
+import java.util.*;
 import java.util.function.Consumer;
 
 import org.junit.jupiter.api.Test;
:::
