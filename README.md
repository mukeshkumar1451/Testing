private <T> void insertData(String collectionName, Class<T> pojoClass, List<T> data) {
    if (data != null && !data.isEmpty()) {
        for (T item : data) {
            try {
                // Modify the item to ensure `lastUpdated` is in the correct format
                String lastUpdatedField = "lastUpdated";
                Object lastUpdatedValue = getFieldValue(item, lastUpdatedField);

                // Convert the lastUpdated value to target format
                String formattedLastUpdated = null;
                if (lastUpdatedValue instanceof String) {
                    String lastUpdatedString = (String) lastUpdatedValue;
                    LocalDateTime localDateTime = LocalDateTime.parse(lastUpdatedString, DATE_FORMATTER);
                    formattedLastUpdated = localDateTime.format(DATE_FORMATTER);
                    setFieldValue(item, lastUpdatedField, formattedLastUpdated);
                }

                // Check if the document already exists in the target collection
                Query query = new Query().addCriteria(Criteria.where("id").is(getFieldValue(item, "id")));
                T existingItem = targetMongoTemplate.findOne(query, pojoClass, collectionName);

                if (existingItem != null) {
                    // Document exists, check if the `lastUpdated` value has changed
                    String existingLastUpdated = (String) getFieldValue(existingItem, lastUpdatedField);

                    if (!formattedLastUpdated.equals(existingLastUpdated)) {
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

            } catch (Exception e) {
                System.err.println("Error processing document: " + e.getMessage());
            }
        }

        System.out.println("Successfully processed " + data.size() + " documents in collection " + collectionName);
    }
}
