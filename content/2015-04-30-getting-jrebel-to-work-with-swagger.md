title: Getting JRebel to work while testing with Swagger
date: Thu Apr 30 2015 13:23:22 GMT-0500 (CDT)
author: Andrew Smith

While installing Swagger I found it very frustrating that I was having to rebuild the container and restart all the time (even with JRebel running).  Also it was very confusing to have to add 4 different servlets to my web.xml in order to get the configs set up correctly.  What follows is what I did to get pure realtime editing with auto deployment using swagger-jersey2 1.5.3-M1

## Create a class that extends ApiListingResource that serves up the swagger.json file

	@Override
	@GET
	@Produces(MediaType.APPLICATION_JSON)
	@Path("/swagger.json")
	public Response getListingJson(
			@Context
			Application app,
			@Context
			ServletConfig sc,
			@Context
			HttpHeaders headers,
			@Context
			UriInfo uriInfo) {

		boolean isDevelopment = true; //TODO: Pull from a config file
		if (!initializedOnce || isDevelopment) {
			setupBootstrap(sc.getServletContext());
		}

		Response resp = super.getListingJson(app, sc, headers, uriInfo);

		if (isDevelopment) {
			resetSwaggerInit();
		}
		initializedOnce = true;
		return resp;
	}


## Install it as a provider in your default jersey servlet

    <init-param>
      <param-name>jersey.config.server.provider.classnames</param-name>
      <param-value>
        com.edatasource.api.analyst.v4.doc.SwaggerDocumentationListing,
        com.wordnik.swagger.jaxrs.listing.SwaggerSerializers
      </param-value>
    </init-param>

## Setup your bootstrap config for swagger awesomeness

One thing to note is that the bean config must be setup because it will override properties set into the swagger object:

	BeanConfig beanConfig = new BeanConfig();
	beanConfig.setTitle("eDatasource V4 API");
	beanConfig.setBasePath(context.getContextPath() + "/v4");
	beanConfig.setContact("my.email@emailsRus.com");
	beanConfig.setVersion("4");

	beanConfig.setDescription("This is an example of how to use swagger to generate API specs.  You can find out more about Swagger " +
			"at <a href=\"http://swagger.io\">http://swagger.io</a>.  For this sample, " +
			"you can use the api key \"XXXX\" to test the authorization filters");

	beanConfig.setResourcePackage(com.edatasource.api.analyst.v4.resource.VolumeResource.class.getPackage().getName());
	beanConfig.setPrettyPrint(true);
	beanConfig.setScan(true);

	Swagger swagger = beanConfig.configure(new Swagger());
	swagger.securityDefinition("api_key", new ApiKeyAuthDefinition("api_key", In.HEADER));

	scanPackagesForTagDescriptions(beanConfig.getResourcePackage(), swagger);

	FilterFactory.setFilter(new AuthV4Filter());

	context.setAttribute("swagger", swagger);

## Make call to "reset" their initialize variable that tells swagger to scan the annotations again

	private void resetSwaggerInit() {
		try {
			Field field = ApiListingResource.class.getDeclaredField("initialized");
			field.setAccessible(true);
			field.set(null, false);
			field.setAccessible(false);
		} catch (Throwable t) {
			logger.error("Unable to reset initialized.", t);
		}
	}

There ya go, enjoy the documentation of swagger with the brilliance of jrebel reloading your documentation for you!

Another minor annoyance I found was with descritions on @Api annotations not doing any thing so I built this annotation scanner to perform the swagger "tag" work for me

	private void scanPackagesForTagDescriptions(String resourcePackage, Swagger swagger) {
		Reflections reflections = new Reflections(
				new ConfigurationBuilder().setUrls(
						ClasspathHelper.forPackage(resourcePackage)).setScanners(new TypeAnnotationsScanner(), new SubTypesScanner()));

		Set<Class<?>> types = reflections.getTypesAnnotatedWith(Api.class);

		for (Class<?> type : types) {
			Api annotation = type.getAnnotation(Api.class);
			String tag = annotation.value().replaceAll("/", "");
			String description = annotation.description();

			swagger.tag(new Tag().name(tag).description(description));

		}
	}
