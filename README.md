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
