Create Custom Entity in ElasticSearch For LR 7.2/DXP
ElasticSearch is now default search engine from Liferay 7/DXP. ElasticSearch is an advanced search engine compared to Lucene which Liferay used in prior versions. Liferay internally uses elasticsearch for searching purpose. for example, full-text search, analytics storage, autocomplete, spell checker, geo-distance etc. Now, we can use all these features in custom entities as well by implementing indexers for the custom entity. Here I’ll show you how you can use elasticsearch for custom entity Liferay 7.2/DXP.

Step 1: Create a custom entity using service builder.
We will implement Elasticsearch functionality for one custom entity called “Deal”. Here I’m using LR’s service builder functionality to create a custom entity.
<entity local-service="true" name="Deal" remote-service="true" uuid="true">

	<column name="dealId" primary="true" type="long" />
	<column name="title" type="String" />
	<column name="description" type="String" />
	<column name="image" type="String" />
	<column name="categoryId" type="long" />
	<column name="price" type="double" />

	<column name="groupId" type="long" />
	<column name="companyId" type="long" />
	<column name="userId" type="long" />
	<column name="userName" type="String" />
	<column name="createDate" type="Date" />
	<column name="modifiedDate" type="Date" />

</entity>

Step 2: Create an Indexer class for the custom entity
It’s best practice to create indexer classes within the same service builder in which we have created an entity. I’ve created a new package in the service builders’ service layer.
Create a new class EntityIndexer that extends BaseIndexer abstract class with Entity name as a type argument.
@Component(immediate = true, service = Indexer.class)
public class DealIndexer extends BaseIndexer<Deal> {
	public static final String CLASS_NAME = DealIndexer.class.getName();
}

Step 3: Implement DealIndexer constructor
Implement constructor in custom indexer class to set default selected field names to retrieve documents from elasticsearch.
It will set permission to indexed the results. If we are not setting this permission to indexer class then Elasticsearch will return all matching documents for this entity regardless of user’s permission to that resource. So it’s better to set permission to an indexer.
public DealIndexer() {
	setDefaultSelectedFieldNames(
		Field.COMPANY_ID, Field.ENTRY_CLASS_NAME, Field.ENTRY_CLASS_PK,
		Field.GROUP_ID, Field.MODIFIED_DATE);
	setPermissionAware(true);
	setFilterSearch(true);
}
Step 4: Override hasPermission Method
Implement hasPermission method of indexer class to make sure that the user has VIEW permission to fetch results from an elasticsearch engine.
@Override
public boolean hasPermission(
        PermissionChecker permissionChecker, String entryClassName, 
        long entryClassPK, String actionId) 
    throws Exception {

    return containsPermissions(
        permissionChecker, entryClassPK, ActionKeys.VIEW);
}
Step 5: Override doGetSummary method
Implement the doGetSummary method to return a summary. Set max content length for summary content’s max length.
@Override protected Summary doGetSummary( Document document, Locale locale, String snippet, PortletRequest portletRequest, PortletResponse portletResponse) { Summary summary = createSummary(document); summary.setMaxContentLength(200); return summary; } 
Step 6: Override doGetDocument method
This is the most important method in the indexer class. It defines which columns we are going to add in elasticsearch. Here we must add only those methods which we will use to retrieve results from the search engine
@Override
protected Document doGetDocument(Deal deal) throws Exception {
	
	Document document = getBaseModelDocument(Deal.class.getName(), deal);
	
	// Add column to be indexed
	document.addNumber("dealId", deal.getDealId());
	document.addNumber("dealPrice", deal.getPrice());
	document.addNumber("dealTitle", deal.getTitle());
	
	return document;
}

Step 7: Override doReindex method
We need to implement 3 doReindex method.
First doReindex method takes a single parameter which is an entity object. This method gets called when the user explicitly called reindex from the control panel.
The second doReindex(String className, long classPK) method takes two arguments. Here, we need to retrieve the deal based on the primary key by calling DealLocalServiceUtil’s getDeal method. Then pass the deal object to the first doReindex method.
Third doReindex(String[] ids) method takes a single argument which is an array of companyIds.
@Override
protected void doReindex(Deal deal) throws Exception {
	Document document = getDocument(deal);
	indexWriterHelper.updateDocument(
			getSearchEngineId(), deal.getCompanyId(), document,
			isCommitImmediately());
}

@Override
protected void doReindex(String className, long classPK) throws Exception {
	Deal deal = DealLocalServiceUtil.getDeal(classPK);
	doReindex(deal);
}

@Override
protected void doReindex(String[] ids) throws Exception {
	// here I’ve just reindex single companyIds documents
	long companyId = GetterUtil.getLong(ids[0]);
	reindexEntries(companyId);
}

Step 8: Override doDelete method
Override doDelete method. This method will delete documents from ElasticSearch once any entity will be deleted from the database.
@Override
protected void doDelete(Deal deal) throws Exception {
	deleteDocument(deal.getCompanyId(), deal.getDealId());
}
That’s it. Your custom entity is ready to be index. Whenever any row is added, updated, or deleted, the search index must be updated accordingly. Once your data will be indexed then you can fetch documents from Elasticsearch using LR’s ElasticSearch APIs.

Note: From LR 7.2 they have changed kernel version so add the following dependency in your build.gradle file.

compileOnly group: "com.liferay.portal", name: "com.liferay.portal.kernel", version: "4.4.0"
