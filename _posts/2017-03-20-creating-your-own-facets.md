---
layout: post
title: Creating your own marketing facets based on promotion status
category: episerver
tags: [episerver, promotions]
---

With the release of Episerver Commerce 10.4.0 a new feature where added so you can filter campaigns by status, but I wanted to filter the promotions by status. In this blogpost I will show you how to deactivte the default filters and how you can make your own.

The result should look like this: 
![promotion status facets]({{ site.baseUrl }}/images/2017-03-20/promotion-status-facets.png)

The [documentation](http://world.episerver.com/documentation/developer-guides/commerce/marketing/custom-facets-in-the-marketing-overview/) for creating your own facets came last week and I started creating my own facets so they who make promotions at my company could filter promotions by status.

Start with creating a new class that inherit from CampaignFacet.
In the constructor,remove the builtin groups with ``` Groups.Clear() ```.

then in the construct add a new group like
```csharp
Groups.Add(facetFactory.CreateFacetGroup(
    CustomFacetConstants.PromotionStatus, // the id of this facetgroup, used later to identify what facets that is choosen
    "Promotion status", // this is the name the editors will see in the cms
    GetPromotionStatusFacetItems(), // here we get FacetItems. This is explained later
    new FacetGroupSettings(
        FacetSelectionType.Multiple, // decides if the user can select multiple or just one facet at a time
        0, // how many facets to show before a "more" option comes
        true, // decides if to show collapse ("more") or not
        true, // if you have icons for the facets, then set this to true
        true, // if true, then a matching number will be set next to the facet  in the cms
        Enumerable.Empty<string>() // if this group is dependent on another group (to get correct matching number)
        )
    )
);
```

The GetPromotionStatusFacetItems() is a simple list of FacetItems

```csharp
private IEnumerable<FacetItem> GetPromotionStatusFacetItems()
{
    return new List<FacetItem>()
    {
        new FacetItem("active", "Active", "epi-statusIndicatorIcon epi-statusIndicator4"),
        new FacetItem("scheduled", "Scheduled", "epi-statusIndicatorIcon epi-statusIndicator6"),
        new FacetItem("expired", "Expired", "epi-statusIndicatorIcon epi-statusIndicator100"),
        new FacetItem("inactive", "Inactive", "epi-statusIndicatorIcon epi-statusIndicator5")
    };
}
```

The last property we send in the FacetItem is css class so we can get icons next to the facet name. I have used here the ones that episerver provide for the "active, scheduled, expired and inactive" for their campaign  status group.

The full class should look like this
```csharp
public class CustomFacet : CampaignFacet
{
    private readonly IContentLoader _contentLoader;
    public CustomFacet(FacetFactory facetFactory, IMarketService marketService, LocalizationService localizationService, IContentLoader contentLoader) : base(facetFactory, marketService, localizationService)
    {
        _contentLoader = contentLoader;

            // clears the builtin facets
        Groups.Clear();

        Groups.Add(facetFactory.CreateFacetGroup(
            CustomFacetConstants.PromotionStatus, // the id of this facetgroup, used later to identify what facets that is choosen
            "Promotion status", // this is the name the editors will see in the cms
            GetPromotionStatusFacetItems(), // here we get FacetItems. This is explained later
            new FacetGroupSettings(
                FacetSelectionType.Multiple, // decides if the user can select multiple or just one facet at a time
                0, // how many facets to show before a "more" option comes
                true, // decides if to show collapse ("more") or not
                true, // if you have icons for the facets, then set this to true
                true, // if true, then a matching number will be set next to the facet  in the cms
                Enumerable.Empty<string>() // if this group is dependent on another group (to get correct matching number)
                )
            )
        );
    }

    private IEnumerable<FacetItem> GetPromotionStatusFacetItems()
    {
        return new List<FacetItem>()
        {
            new FacetItem("active", "Active", "epi-statusIndicatorIcon epi-statusIndicator4"),
            new FacetItem("scheduled", "Scheduled", "epi-statusIndicatorIcon epi-statusIndicator6"),
            new FacetItem("expired", "Expired", "epi-statusIndicatorIcon epi-statusIndicator100"),
            new FacetItem("inactive", "Inactive", "epi-statusIndicatorIcon epi-statusIndicator5")
        };
    }
}
```

The next thing that needs to be done is to make a new RestStore class that we will use to serve the new facets to the cms.

```csharp
[RestStore("customfacet")]
public class CustomFacetStore : RestControllerBase
{
    private readonly FacetFactory _facetFactory;
    private readonly IMarketService _marketService;
    private readonly LocalizationService _localizationService;
    private readonly IContentLoader _contentLoader;
    private readonly IEnumerable<GetContentsByFacet> _filters;

    public CustomFacetStore(FacetFactory facetFactory, IMarketService marketService, LocalizationService localizationService, IContentLoader contentLoader)
    {
        _facetFactory = facetFactory;
        _marketService = marketService;
        _localizationService = localizationService;
        _contentLoader = contentLoader;

        _filters = new GetContentsByFacet[]
        {
            new GetPromotionsByStatus(_contentLoader)
        };
    }

    public RestResult Get(string id, string facetString, ContentReference parentLink)
    {
        var customFacet = new CustomFacet(_facetFactory, _marketService, _localizationService, _contentLoader);
        var facetQueryHandler = new FacetQueryHandler();
        facetQueryHandler.CalculateMatchingNumbers(
            _contentLoader.GetChildren<IContent>(parentLink),
            customFacet.Groups,
            facetString,
            _filters
        );

        return Rest(customFacet);
    }
}
```

If you earlier set the "matching number" property to false, then you can leave out the _filters, and in the Get method just return the CustomFacet.

To get matching numbers working, create a new class that inherits from "GetContentsByFacet" and set the key to the same as the GroupName, in our case it is in the "CustomFacetConstants.PromotionStatus" the complete class looked like this for me

```csharp
public class GetPromotionsByStatus : GetContentsByFacet
{
    private readonly IContentLoader _contentLoader;
    public GetPromotionsByStatus(IContentLoader contentLoader)
    {
        _contentLoader = contentLoader;
    }

    public override IEnumerable<IContent> GetItems(IEnumerable<IContent> items, IEnumerable<string> facets)
    {
        return items.SelectMany(
            x => _contentLoader
            .GetChildren<PromotionData>(x.ContentLink)
            .Where(y => AvailableFor(y, facets)));
            
    }

    public bool AvailableFor(PromotionData promotion, IEnumerable<string> facets)
    {
        var isAvailable = false;
        foreach (var facet in facets)
        {
            switch (facet)
            {
                case "active":
                    isAvailable = promotion.IsActive && !IsExpired(promotion) && !IsScheduled(promotion);
                    break;
                case "inactive":
                    isAvailable = !promotion.IsActive && !IsExpired(promotion) && !IsScheduled(promotion); ;
                    break;
                case "expired":
                    isAvailable = IsExpired(promotion);
                    break;
                case "scheduled":
                    isAvailable = IsScheduled(promotion);
                    break;
            }
        }

        return isAvailable;
    }

    private bool IsScheduled(PromotionData promotion)
    {
        return promotion.Schedule.ValidFrom != DateTime.MinValue && promotion.Schedule.ValidFrom > DateTime.Now;
    }

    private bool IsExpired(PromotionData promotion)
    {
        return promotion.Schedule.ValidUntil != DateTime.MinValue && promotion.Schedule.ValidUntil < DateTime.Now;
    }

    public override string Key => CustomFacetConstants.PromotionStatus;
}
```

Here we go through all the promotions for each SalesCampaign so we can return total of matching promotions. Episerver will make one request per facet we have to get the correct number for the matching element.

To be able to use the new reststore in the cms, we will need to overwrite the builtin restore. Edit your module initalizer so it looks something like this

```js
define([
    "dojo/_base/declare",
    "epi/_Module",
    "epi/routes"
], function (
    declare,
    _Module,
    routes
) {
    return declare([_Module], {
        initialize: function () {
            this.inherited(arguments);

            var registry = this.resolveDependency("epi.storeregistry");
            // remove existing facet store
            if (registry.get("epi.commerce.facet")) {
                delete registry._stores["epi.commerce.facet"];
            }
            // register the custom facet
            registry.create("epi.commerce.facet", routes.getRestPath({ moduleArea: "app", storeName: "customfacet" }));
        }
    });
});
```

If you try to build the solution now, then you should be able to see your facets in the marketing tab, but if you try to click on one of them then it will not work.

To get the filters to work we need to make a couple of more classes. We need to make one class that will fetch all the campaigns and their promotions, and one to tell to use that class to get campaign and promotions.

Create a class that inherits from GetContentsByFacet. This will look very much like the previous GetContentsByFacet we made earlier

```csharp
public class GetCampaignsByPromotionStatus : GetContentsByFacet
{
    private readonly IContentLoader _contentLoader;

    public GetCampaignsByPromotionStatus(IContentLoader contentLoader)
    {
        _contentLoader = contentLoader;
    }

    public override IEnumerable<IContent> GetItems(IEnumerable<IContent> items, IEnumerable<string> facets)
    {
        return items.Where(x => CheckStatus(x,  facets));
    }

    public bool CheckStatus(IContent content, IEnumerable<string> facets)
    {
        if (content is SalesCampaign)
        {
            return CampaignHavePromotions((SalesCampaign) content, facets);
        }

        if (content is PromotionData)
        {
            return AvailableFor((PromotionData) content, facets);
        }

        return false;
    }

    public bool CampaignHavePromotions(SalesCampaign campaign, IEnumerable<string> facets)
    {
        return _contentLoader.GetChildren<PromotionData>(campaign.ContentLink)
            .Any(x => AvailableFor(x, facets));
    }

    public bool AvailableFor(PromotionData promotion, IEnumerable<string> facets)
    {
        var isAvailable = false;
        foreach (var facet in facets)
        {
            switch (facet)
            {
                case "active":
                    isAvailable = promotion.IsActive && !IsExpired(promotion) && !IsScheduled(promotion);
                    break;
                case "inactive":
                    isAvailable = !promotion.IsActive && !IsExpired(promotion) && !IsScheduled(promotion);
                    break;
                case "expired":
                    isAvailable = IsExpired(promotion);
                    break;
                case "scheduled":
                    isAvailable = IsScheduled(promotion);
                    break;
            }

            if (isAvailable)
                return true; // return true right away when we get a hit, else we might get a false false
        }

        return false;
    }

    private bool IsScheduled(PromotionData promotion)
    {
        return promotion.Schedule.ValidFrom != DateTime.MinValue && promotion.Schedule.ValidFrom > DateTime.Now;
    }

    private bool IsExpired(PromotionData promotion)
    {
        return promotion.Schedule.ValidUntil != DateTime.MinValue && promotion.Schedule.ValidUntil < DateTime.Now;
    }

    public override string Key => CustomFacetConstants.PromotionStatus;
}
```

The biggest diffrence is that episerver will push both SalesCampaign and promotionsdata IContents, while the previous GetContentByFacet for matching numbers only sendt Campaigns and for that reason we had to get all children for the SalesCampaign to get the correct matching number. 

Episerver will use the IContents we return from this new class to show campaigns/promotions.

To tell episerver to use this new class to get SalesCampaign/Promotions we need to make a new class that inherits from GetSalesCampaignChildrenQuery and have an class attribute of ```csharp [ServiceConfiguration(typeof(IContentQuery))] ``` so episerver know to use this class to get SalesCampaigns/promotions when quering

```csharp
[ServiceConfiguration(typeof(IContentQuery))]
public class CustomGetSalesCampaignChildrenQuery : GetSalesCampaignChildrenQuery
{
    public CustomGetSalesCampaignChildrenQuery(
        IContentQueryHelper queryHelper,
        IContentRepository contentRepository,
        LanguageSelectorFactory languageSelectorFactory,
        CampaignInfoExtractor campaignInfoExtractor,
        FacetQueryHandler facetQueryHandler)
    : base(queryHelper, contentRepository, languageSelectorFactory, campaignInfoExtractor, facetQueryHandler) { }

    public override int Rank => 1000; // this needs to be higher then GetSalesCampaignChildrenQuery to work

    protected override IEnumerable<GetContentsByFacet> FacetFunctions => new GetContentsByFacet[] {
        new GetCampaignsByPromotionStatus(_contentRepository), 
    };
}
```

And thats it :)

I have made an example with the quicksilver repository which can be found [here](https://github.com/Sebbe/Quicksilver/tree/CustomMarketingFacets). I have also added one more Facet group so the editors can filter on campaign name
