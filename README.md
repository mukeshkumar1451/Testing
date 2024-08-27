import org.springframework.data.mongodb.core.query.Update;
import java.time.ZonedDateTime;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.lang.reflect.Field;
import java.util.List;

private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
private static final ZoneId TIME_ZONE = ZoneId.of("Asia/Kolkata");

private <T> void insertData(String collectionName, Class<T> pojoClass, List<T> data) {
    if (data != null && !data.isEmpty()) {
        for (T item : data) {
            try {
                // Extract the ID and lastUpdated fields from the item
                String idField = "_id"; // Assuming the ID field is named "_id"
                String lastUpdatedField = "lastUpdated";
                
                Object idValue = getFieldValue(item, idField);
                Object lastUpdatedValue = getFieldValue(item, lastUpdatedField);

                // Convert lastUpdated to the target format
                String formattedLastUpdated = null;
                if (lastUpdatedValue instanceof String) {
                    String lastUpdatedString = (String) lastUpdatedValue;
                    ZonedDateTime localDateTime = ZonedDateTime.parse(lastUpdatedString, DATE_FORMATTER.withZone(TIME_ZONE));
                    formattedLastUpdated = localDateTime.withZoneSameInstant(TIME_ZONE).format(DATE_FORMATTER);
                    setFieldValue(item, lastUpdatedField, formattedLastUpdated);
                }

                // Check if the document already exists in the target collection
                Query query = new Query(Criteria.where(idField).is(idValue));
                T existingItem = targetMongoTemplate.findOne(query, pojoClass, collectionName);

                if (existingItem != null) {
                    // Document exists, compare lastUpdated values
                    String existingLastUpdated = (String) getFieldValue(existingItem, lastUpdatedField);
                    ZonedDateTime existingDateTime = ZonedDateTime.parse(existingLastUpdated, DATE_FORMATTER.withZone(TIME_ZONE));
                    ZonedDateTime newDateTime = ZonedDateTime.parse(formattedLastUpdated, DATE_FORMATTER.withZone(TIME_ZONE));

                    if (newDateTime.isAfter(existingDateTime)) {
                        // Update the entire document with the new values
                        Update update = new Update();
                        for (Field field : item.getClass().getDeclaredFields()) {
                            field.setAccessible(true);
                            Object value = field.get(item);
                            update.set(field.getName(), value);
                        }
                        targetMongoTemplate.updateFirst(query, update, collectionName);
                        System.out.println("Document updated in collection " + collectionName);
                    } else {
                        System.out.println("No change in `lastUpdated` value for document in collection " + collectionName);
                    }
                } else {
                    // Insert new document if it doesn't exist
                    targetMongoTemplate.insert(item, collectionName);
                    System.out.println("Document inserted into collection " + collectionName);
                }

            } catch (NoSuchFieldException e) {
                System.err.println("Error processing document: Field not found - " + e.getMessage());
            } catch (IllegalAccessException e) {
                System.err.println("Error processing document: Illegal access - " + e.getMessage());
            } catch (Exception e) {
                System.err.println("Error processing document: " + e.getMessage());
            }
        }

        System.out.println("Successfully processed " + data.size() + " documents in collection " + collectionName);
    }
}

private Object getFieldValue(Object item, String fieldName) throws NoSuchFieldException, IllegalAccessException {
    Field field = item.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    return field.get(item);
}

private void setFieldValue(Object item, String fieldName, Object value) throws NoSuchFieldException, IllegalAccessException {
    Field field = item.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    field.set(item, value);
}
