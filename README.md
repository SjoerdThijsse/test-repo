```java
/**
 * Default endpoint to show all the available collections to execute.
 * 
 * @return The `export_collections` page filled with objects.
 */
@RequestMapping(method = RequestMethod.GET)
public ModelAndView collections() {
	ModelAndView model = new ModelAndView("export_collectionss");
	
	// Get a list of all the available collections to execute.
	List<Collection> collections = collectionDAO.getAll();
	CollectionsForm collectionsForm = new CollectionsForm(collections);
	
	// Set the form holder.
	model.addObject("collectionsForm", collectionsForm);
	logger.info("Fetched " + Integer.toString(collections.size()) + " results.");
	
	// Enable or disable the export button based on the value;
	model.addObject("isBusy", ExportController.isBusy);
	
	return model;
}
```
