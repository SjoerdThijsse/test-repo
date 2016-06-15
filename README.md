# test-repo

this is a test-repo to pull seperate folders from git.


```java
/**
 *
 * @author Sjoerd Thijsse
 *
 */
@Entity(name = "collection")
public class Collection implements Serializable, Comparable<Collection> {

    @JsonIgnore
    private static final long serialVersionUID = 368184984178588300L;

    @Id
    @Column(name = "collection_name")
    @JsonProperty("collection_name")
    private String collectionName;

    @Column(name = "collection_type")
    @JsonProperty("collection_type")
    private String collectionType;

    @Transient
    @JsonProperty("properties_last_edited")
    private Date propertiesLastEdited;

    @Transient
    @JsonProperty("eml_last_edited")
    private Date emlLastEdited;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "domain_id")
    @JsonIgnore
    private Domain domain;

    @OneToOne(mappedBy = "collection", fetch = FetchType.EAGER, orphanRemoval = true)
    @JsonIgnore
    private Task task;
    
}
```

```java
/**
 * Execute the export of a list of collections, which are nested in the
 * results, asynchronously.
 *
 * @param results
 *            The results which will need to be processed.
 */
public void asyncExport(final List<Result> results) {
    if (!isBusy) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    isBusy = true;

                    for (Result result : results) {
                        Date now = new Date();

                        Task task = result.getTask();
                        task.setLastExecuted(now);
                        task = taskDAO.saveOrUpdate(task);

                        result.setStatus("Bezig");
                        result.setStartDate(now);
                        result.setTask(task);
                        result = resultDAO.saveOrUpdate(result);

                        String[] collection = new String[] { result
                                .getTask().getCollection()
                                .getCollectionName() };
                        ExportDwCAUtilities.setLogFileName(result
                                .getLogFile());

                        ExportExecutor.executeFuture(collection);
                    }
                } catch (InterruptedException | PersistenceException | ExecutionException e) {
                    e.printStackTrace();
                } finally {
                    isBusy = false;
                }
            }
        }).start();
    }
}
```

```java
/**
 * Endpoint for setting a result as done.
 *
 * @param resultMessage
 *            The body send by Logstash to the server.
 * @return The new results in json format.
 */
@RequestMapping(value = "/results/done", method = RequestMethod.POST)
public ResponseEntity<Result> done(@RequestBody ResultMessage resultMessage) {
    logger.info("result is done");
    try {
        // Get the file name of the log file.
        File path = new File(resultMessage.getPath());
        String logFile = path.getName();

        // Find the result object based on the file name of the log file.
        Result result = resultDAO.getByLogFile(logFile);

        if (result != null) {
            // Set the new values of the found result object.
            Date now = new Date();
            long duration = now.getTime() - result.getStartDate().getTime();
                
            result.setEndDate(now);
            result.setStatus("Uitgevoerd");
            result.setDuration(new Time(duration));
            result.setOccurrences(resultMessage.getOccurrences());

            // Update the result with the new values.
            result = resultDAO.saveOrUpdate(result);
            return new ResponseEntity<Result>(result, HttpStatus.OK);
        } else {
            return new ResponseEntity<>(HttpStatus.INTERNAL_SERVER_ERROR);
        }
    } catch (PersistenceException e) {
        e.printStackTrace();
        return new ResponseEntity<>(HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

```java
/**
 * Download a given file.
 * @param file The file to download
 * @param res The response which will be sent by the server.
 * @return A response to download a file.
 */
private static ResponseEntity<InputStreamResource> downloadFile(File file,
                                                            HttpServletResponse res) {
    logger.info("Downloading file: '" + file.getName() + "'.");
   InputStreamResource isr = null;
   try {
        res.setContentType("text/plain; charset=UTF-8");
        res.setHeader("Content-Disposition",
                "attachment;filename=" + file.getName());
        isr = new InputStreamResource(new FileInputStream(file));
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
    return new ResponseEntity<InputStreamResource>(isr, HttpStatus.OK);
}
```

```java
/**
 * Endpoint for updating the collection property and eml files from the
 * Github repository.
 *
 * @return A view with the updated collections.
 */
@RequestMapping(value = "/update", method = RequestMethod.POST)
public ModelAndView updateAll() {
	try {
		Repository repo = new FileRepositoryBuilder().setGitDir(
				new File(ExportDwCAUtilities.getCollectionConfigDir()
						+ "/.git")).build();

		Git git = new Git(repo);
		git.fetch().setRemote("origin").call();
		git.checkout().setStartPoint("refs/remotes/origin/test")
				.addPath("collections/*").setAllPaths(true).call();

		git.close();

		startupController.addCollections();
	} catch (IOException | GitAPIException e) {
		e.printStackTrace();
	}

	return new ModelAndView("redirect:/export/collections");
}
```

```
input {
  file {
    path => "/home/sjoerd/Software/nba/export/log/*.log"
    exclude => "dwca-exporter*.log"
  }
}

filter {
  grok {
    break_on_match => false
    match => { "message" => [ "%{TIMESTAMP_ISO8601}  %{LOGLEVEL} %{GREEDYDATA}
                               %{JAVACLASS} %{SPACE} %{DATA} %{INT:occurrences}
                               %{GREEDYDATA} %{TIME:duration} %{GREEDYDATA}
                               %{QS:collectionName}" ] }
    remove_tag => [ "_frokparsefailure" ]
  }

  if "_grokparsefailure" in [tags] { drop {} }
}

output {
  http {
    url => "http://127.0.0.1:8080/nl.naturalis.nba.admin/export/results/done"
    http_method => "post"
    content_type => "application/json"
    automatic_retries => 5
  }

  file { path => "/home/sjoerd/test.log" }
  stdout { codec => json }
}
```
```grok
Exported 274263 records to occurrence file in 00:01:10 time for collectionName 'aves'
%{DATA} %{INT:occurrences} %{GREEDYDATA} %{TIME:duration} %{GREEDYDATA} %{QS:collectionName}
```
