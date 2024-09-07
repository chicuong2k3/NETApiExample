# .NET Core API

## REST

- REST is an architectural style.
- REST is not a standard.
- REST is protocol agnostic.

- Should know 6 constraints of REST.
- Richardson Maturity Model

## Method Safety and Method Idempotency

[Routing](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-8.0)

## Content Negotiation (Output Formatters)

- Content negotiation is the process of selecting the best representation 
for a given response when there are multiple representations available.

- If the Accept header is not present, the server should return the default representation.
- If the Accept header is present and the media type is not supported, the server should return 406 Status Code.

```c#
builder.Services.AddControllers(config =>
{
    config.RespectBrowserAcceptHeader = true;
    config.ReturnHttpNotAcceptable = true;

    //config.OutputFormatters.Add(new CsvOutputFormatter());

})
.AddXmlDataContractSerializerFormatters();
```

### Vender-specific Media Types - Semantic Media Types

Example: application/vnd.company.hateoas+json
Example: application/vnd.company.book.friendly.hateoas+json
Example: application/vnd.company.book.friendly+json
Example: application/vnd.company.book.full.hateoas+json
Example: application/vnd.company.book.full+json

Format: top-level-type/vendor-prefix.vendor-id.media-type-name+suffix

```c#
public IActionResult GetBook(Guid id, [FromHeader(Name = "Accept")] string? mediaType)
{
	if (!MediaTypeHeaderValue.TryParse(mediaType, out var parsedMediaType))
	{
		return BadRequest();
	}


    var includeLinks = parsedMediaType.SubTypeWithoutSuffix
                                .EndsWith("hateoas", StringComparison.InvariantCultureIgnoreCase);
    
    // if includeLinks is true, include links in the response

    var primaryMediaType = includeLinks ? parsedMediaType.SubTypeWithoutSuffix
								.Substring(0, parsedMediaType.SubTypeWithoutSuffix.Length - 8)
                                : parsedMediaType.SubTypeWithoutSuffix;

    // if (primaryMediaType != "application/vnd.company.book.full")

	return Ok();
}
```

```c#
builder.Services.Configure<MvcOptions>(config =>
{
    var newtonsoftJsonOutputFormatter = config.OutputFormatters
		.OfType<NewtonsoftJsonOutputFormatter>()
		?.FirstOrDefault();

    if (newtonsoftJsonOutputFormatter is not null) 
    {
        newtonsoftJsonOutputFormatter.SupportedMediaTypes.Add("application/vnd.company.hateoas+json");
	}
    
	config.OutputFormatters.Add(new CsvOutputFormatter());
});
```

### Action Constraints

```c#
[AttributeUsage(AttributeTargets.All, Inherited = true, AllowMultiple = false)]
public class RequestHeaderMatchesMediaTypeAttribute : Attribute, IActionConstraint
{
	private readonly string _requestHeaderToMatch;
	private readonly MediaTypeCollection _mediaTypes = new();

	public RequestHeaderMediaTypeAttribute(
		string requestHeaderToMatch, 
		string mediaType,
		params string[] otherMediaTypes)
	{
		_requestHeaderToMatch = requestHeaderToMatch ?? throw new ArgumentNullException(nameof(requestHeaderToMatch));
		
		if (MediaTypeHeaderValue.TryParse(mediaType, out var parsedMediaType))
		{
			_mediaTypes.Add(parsedMediaType);
		}
		else 		
		{
			throw new ArgumentException("Invalid media type.", nameof(mediaType));
		}
		
		foreach (var otherMediaType in otherMediaTypes)
		{
			if (MediaTypeHeaderValue.TryParse(mediaType, out var parsedOtherMediaType))
			{
				_mediaTypes.Add(parsedOtherMediaType);
			}
			else 		
			{
				throw new ArgumentException("Invalid media type.", nameof(otherMediaType));
			}
		}
	}

	public int Order { get; }

	public bool Accept(ActionConstraintContext context)
	{
		var requestHeaders = context.RouteContext.HttpContext.Request.Headers;
		if (!requestHeaders.ContainsKey(_requestHeaderToMatch))
		{
			return false;
		}

		var parsedRequestMediaType = new MediaType(requestHeaders[_requestHeaderToMatch]);
		
		foeach (var mediaType in _mediaTypes)
		{
			var parsedMediaType = new MediaType(mediaType);
			
			if (parsedRequestMediaType.Equals(parsedMediaType))
			{
				return true;
			}
		}

		return fase;

	}
}
```

```c#
[RequestHeaderMediaType("Content-Type", "application/json")]
```


## Custom Validation

```c#
builder.Services.AddControllers(config =>
{
    config.RespectBrowserAcceptHeader = true;
    config.ReturnHttpNotAcceptable = true;

    //config.OutputFormatters.Add(new CsvOutputFormatter());

})
.AddXmlDataContractSerializerFormatters()
.ConfigureApiBehaviourOptions(setupAction => 
{
    setupAction.InvalidModelStateResponseFactory = context =>
	{
        var problemDetailsFactory = context.HttpContext.RequestServices
                                            .GetRequiredService<ProblemDetailsFactory>();

        var problemDetails = problemDetailsFactory.CreateValidationProblemDetails(
			                                                    context.HttpContext,
			                                                    context.ModelState
                                                             );
       
        problemDetails.Detail = "See the errors field for details.";
        problemDetails.Instance = context.HttpContext.Request.Path;
        problemDetails.Type = "https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/422";
        problemDetails.Status = StatusCodes.Status422UnprocessableEntity;
        problemDetails.Title = "One or more validation errors occurred.";

		problemDetails.Extensions.Add("traceId", context.HttpContext.TraceIdentifier);

		return new UnprocessableEntityObjectResult(problemDetails)
		{
			ContentTypes = { "application/problem+json" }
		};
	};
});
```

### Class Level Validation

```c#
public class Book : IValidatableObject
{
	public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
	{
		if (PublishedAt > DateTime.UtcNow) 
        {
            yeild return new ValidationResult(
                "PublishedAt must be a past date.", 
                new[] { nameof(PublishedAt) }
            );
        }
	}

    [Required]
    public DateTime PublishedAt { get; set; }

}
```

### Custom Validation Attribute

```c#
public sealed class CustomValidationAttribute : ValidationAttribute
{
	protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
	{
		if (value is null || validationContext.ObjectInstance is not BookDto book)
		{
			return new ValidationResult("The field is required.", new [] { nameof(BookDto) });
		}

		return ValidationResult.Success;
	}
}
```

### Validation at Controller Actions

## Custom Model Binders

```c#
public class ListModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        if (!bindingContext.ModelMetadata.IsEnumerableType)
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return Task.CompletedTask;
        }

        var providedValue = bindingContext.ValueProvider.GetValue(bindingContext.ModelName).ToString();
        if (string.IsNullOrEmpty(providedValue))
        {
            bindingContext.Result = ModelBindingResult.Success(null);
            return Task.CompletedTask;
        }

        var genericType = bindingContext.ModelType.GetTypeInfo().GenericTypeArguments[0];
        var converter = TypeDescriptor.GetConverter(genericType);

        var objectArray = providedValue.Split(new[] { "," }, StringSplitOptions.RemoveEmptyEntries)
                                     .Select(x => converter.ConvertFromString(x.Trim()))
                                     .ToArray();
        var typedArray = Array.CreateInstance(genericType, objectArray.Length);

        objectArray.CopyTo(typedArray, 0);

        bindingContext.Model = typedArray;
        bindingContext.Result = ModelBindingResult.Success(bindingContext.Model);
        return Task.CompletedTask;
    }
}
```

```c#
public IActionResult GetAll([ModelBinder(BinderType = typeof(ListModelBinder))] IEnumerable<Guid> ids)
{
	return Ok();
}
```

## PATCH

Must install package Microsoft.AspNetCore.Mvc.NewtonsoftJson

```c#
builder.Services.AddControllers(config =>
{
    config.RespectBrowserAcceptHeader = true;
    config.ReturnHttpNotAcceptable = true;

    //config.OutputFormatters.Add(new CsvOutputFormatter());

})
.AddNewtonsoftJson(setupAction => 
{
    setupAction.SerizalizerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver();
})
.AddXmlDataContractSerializerFormatters(); // this should come after because the output formatter order is matter
```

```c#
```

```c#
public async Task<IActionResult> PartiallyUpdate(
    Guid bookId, 
    JsonPatchDocument<BookRequest> patchDocument)
{
    var bookToPatch = await _bookRepository.GetBookAsync(bookId);
    if (bookToPatch == null)
	{
		return NotFound();
	}

    patchDocument.ApplyTo(bookToPatch, ModelState);

    if (!TryValidateModel(bookToPatch))
    {
        return ValidationProblem(ModelState);
    }

    // update the book in the database
    // ...

    return NocContent();
}
```

## Caching

### Cache Types

- Client Cache (Private Cache) lives on the client
- Gateway Cache/Reverse Proxy Cache (Shared Cache) lives on the server
- Proxy Cache (Shared Cache) lives on the network

### Response Cache

```c#
[ResponseCache(Duration = 60)]
[ResponseCache(CacheProfileName = "240SecondsCacheProfile")]
```

```c#
builder.Services.AddControllers(config => 
{
	config.CacheProfiles.Add("240SecondsCacheProfile", new CacheProfile
	{
		Duration = 240
	});
});

builder.Services.AddResponseCaching();

var app = builder.Build();

app.UseResponseCaching();
```

### Expiration Model

- Cache-Control: max-age=60
- Expires: Thu, 01 Jan 1970 00:00:00 GMT 
	- -> Clocks must be synchronized
	- Offer little control
	- -> Not recommended


- Private Cache:
	- Reduces bandwidth
	- Less requests to the server
- Public Cache:
	- Doesn't reduce bandwidth between the client and the server
	- Reduces requests to the API

### Validation Model (not supported by ResponseCache attribute and middleware)

- Strong Validators: 
	- Change if body or request headers of a response change
	- Can be used in any context (equality is guaranteed)
	- **Revalidate ETag by using If-None-Match header**
	- Example: ETag: "12323"
- Weak Validators: 
	- Does not always change when the response changes (it's up to the server decide when to change)
	- Example: Last-Modified: Thu, 01 Jan 1970 00:00:00 GMT, ETag: "w/12323"


### Cache-Control Directives

- Response:
	- Cache type: public, private
	- Freshness: max-age, s-maxage
	- Validation: 
		- no-cache: the response should not be used to satisfy a subsequent request without successful revalidation with the origin server
		- must-revalidate: if a response become expired, it must be revalidated
		- proxy-revalidate: same as must-revalidate but only for shared caches
	- Vary: Accept, Accept-Language, Accept-Encoding
	- Others:
		- no-store: the response should not be stored in any cache
		- no-transform: the media type of the response should not be transformed

- Request:
	- Freshness: max-age, min-fresh, max-stale
	- Validation: no-cache
	- Others: no-store, no-transform, only-if-cached


### Supporting ETags

Install package Marvin.Cache.Headers

```c#
builder.Services.AddHttpCacheHeaders();

var app = builder.Build();

// go after the UseResponseCaching
app.UseHttpCacheHeaders(expirationModelOptions => 
{
	expirationModelOptions.MaxAge = 60;
	expirationModelOptions.CacheLocation = Marvin.Cache.Headers.CacheLocation.Private;

}, validationModelOptions => 
{
	validationModelOptions.ProxyRevalidate = true;
});
```

**Resource Level**

```c#
[HttpCacheExpiration(CacheLocation = CacheLocation.Public, MaxAge = 60)]
[HttpCacheValidation(MustRevalidate = true)]
```

**For complicate scenarios, don't use Microsoft.AspNetCore.ResponseCaching**

### Popular Shared Cache Servers

- Varnish
- Squid
- Apache Traffic Server

### Popular CDNs

- Azure CDN
- Cloudflare
- Akamai

### Invalidate Cache

## Concurrent 

## Pessimistic Concurrency

Resource is locked. while it's locked, it cannot be modified by other users.

This is not possible in REST because REST is stateless.

## Optimistic Concurrency

Token is returned together with the resource. 
The update can happen as long as the token is valid. ETags are used as validation tokens.

Install package Marvin.Cache.Headers

- Send ETag in If-Match header
- On mismatch, return 412 Precondition Failed
	

## Mapping Service

```c#
public class PropertyMappingValue
{
	public IEnumerable<string> DestinationProperties { get; private set; }
	public bool Revert { get; private set; }

	public PropertyMappingValue(IEnumerable<string> destinationProperties, bool revert = false)
	{
		DestinationProperties = destinationProperties ?? throw new ArgumentNullException(nameof(destinationProperties));
		Revert = revert;
	}
}

public class PropertMappingService : IPropertyMappingService
{
    private readonly Dictionary<string, PropertyMappingValue> _authorPropertyMapping =
		new Dictionary<string, PropertyMappingValue>(StringComparer.OrdinalIgnoreCase)
		{
			{ "Id", new PropertyMappingValue(new List<string> { "Id" }) },
			{ "Genre", new PropertyMappingValue(new List<string> { "Genre" }) },
			{ "Age", new PropertyMappingValue(new List<string> { "DateOfBirth" }, true) },
			{ "Name", new PropertyMappingValue(new List<string> { "FirstName", "LastName" }) }
		};

	private readonly IList<IPropertyMapping> _propertyMappings = new List<IPropertyMapping>();

	public PropertMappingService()
	{
		_propertyMappings.Add(new PropertyMapping<AuthorDto, Author>(_authorPropertyMapping));
	}

	public Dictionary<string, PropertyMappingValue> GetPropertyMapping<TSource, TDestination>()
	{
		var matchingMapping = _propertyMappings.OfType<PropertyMapping<TSource, TDestination>>();
		if (matchingMapping.Count() == 1)
		{
			return matchingMapping.First().MappingDictionary;
		}

		throw new Exception($"Cannot find exact property mapping instance for <{typeof(TSource)}, {typeof(TDestination)}>");
	}
}

public interface IPropertyMapping
{
}
public class PropertMapping<TSource, TDestination> : IPropertyMapping
{
    public Dictionary<string, PropertyMappingValue> MappingDictionary { get; private set; }

    public PropertMapping(Dictionary<string, PropertyMappingValue> mappingDictionary)
	{
		MappingDictionary = mappingDictionary ?? throw new ArgumentNullException(nameof(mappingDictionary));
	}
}
```