import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
private static final ZoneId TIME_ZONE = ZoneId.of("Asia/Kolkata");

private <T> void insertData(String collectionName, Class<T> pojoClass, List<T> data) {
    if (data != null && !data.isEmpty()) {
        for (T item : data) {
            try {
                // Modify the item to ensure `lastUpdated` is in the correct format
                String lastUpdatedField = "lastUpdated";
                Object lastUpdatedValue = getFieldValue(item, lastUpdatedField);

                String formattedLastUpdated = null;
                if (lastUpdatedValue instanceof String) {
                    String lastUpdatedString = (String) lastUpdatedValue;
                    // Parse and convert the `lastUpdated` value
                    ZonedDateTime localDateTime = ZonedDateTime.parse(lastUpdatedString, DATE_FORMATTER.withZone(TIME_ZONE));
                    formattedLastUpdated = localDateTime.withZoneSameInstant(TIME_ZONE).format(DATE_FORMATTER);
                    setFieldValue(item, lastUpdatedField, formattedLastUpdated);
                }

                // Check if the document already exists in the target collection
                Query query = new Query().addCriteria(Criteria.where("_id").is(getFieldValue(item, "_id")));
                T existingItem = targetMongoTemplate.findOne(query, pojoClass, collectionName);

                if (existingItem != null) {
                    // Document exists, check if the `lastUpdated` value has changed
                    String existingLastUpdated = (String) getFieldValue(existingItem, lastUpdatedField);
                    ZonedDateTime existingDateTime = ZonedDateTime.parse(existingLastUpdated, DATE_FORMATTER.withZone(TIME_ZONE));
                    ZonedDateTime newDateTime = ZonedDateTime.parse(formattedLastUpdated, DATE_FORMATTER.withZone(TIME_ZONE));

                    if (newDateTime.isAfter(existingDateTime)) {
                        // Update the existing document with the new `lastUpdated` value
                        Update update = new Update().set(lastUpdatedField, formattedLastUpdated);
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
