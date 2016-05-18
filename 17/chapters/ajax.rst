.. _ajax-label:

.. note::
	A newer version of this tutorial using Django 1.9 is available from `Leanpub: https://leanpub.com/tangowithdjango19 <https://leanpub.com/tangowithdjango19>`_


AJAX in Django with JQuery
==========================
AJAX essentially is a combination of technologies that are integrated together to reduce the number of page loads. Instead of reloading the full page, only part of the page or the data in the page is reloaded. 	If you haven't used AJAX before or would like to know more about it before using it, check out the resources at the Mozilla website: https://developer.mozilla.org/en-US/docs/AJAX

To simplify the AJAX requests, we will be using the JQuery library. Note that if you are using the Twitter CSS Bootstrap toolkit then JQuery will already be added in. Otherwise, download the latest version of JQuery and include it within your application (see Chapter ..).

AJAX based Functionality
------------------------
To make the interaction with the Rango application more seamless let's add in a number of features that use AJAX, such as:

* Add a "Like Button" to let registered users "like" a particular category
* Add inline category suggestions - so that when a user types they can quickly find a category
* Add an "Add Button" to let registered users quickly and easily add a Page to the Category


Create a new file, called ``rango-ajax.js`` and add it to your ``js`` directory. Then in your *base* template include:

.. code-block:: html
	
	<script src="{% static "js/jquery.js" %}"></script>
	<script src="{% static "js/rango-ajax.js" %}"></script>


Here we assume you have downloaded a version of the JQuery library, but you can also just directly refer to it:

.. code-block:: html

	<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js"></script>


Now that the pre-reqs for using JQuery are in place we can use it to pimp the rango application.

Add a "Like Button" 
--------------------
It would be nice to let user, who are registered, denote that they "like" a particular category. In the following workflow, we will let users "like" categories, but we will not be keeping track of what categories they have "liked", we'll be trusting them not to click the like button multiple times.

Workflow
........

To let users "like" certain categories undertake the following workflow:

* In the ``category.html`` template:
	- Add in a "Like" button with ``id="like"``.
	- Add in a template tag to display the number of likes: ``{{% category.likes %}}``
	- Place this inside a div with ``id="like_count"``, i.e. ``<div id="like_count">{{ category.likes }} </div>``
	- This sets up the template to capture likes and to display likes for the category.
	- Note, since the ``category()`` view passes a reference to the category object, we can use that to access the number of likes, with ``{{ category.likes }}`` in the template

* Create a view called, ``like_category`` which will examine the request and pick out the ``category_id`` and then increment the number of likes for that category.
	- Don't forgot to add in the url mapping; i.e  map the ``like_category`` view to ``rango/like_category/``. The GET request will then be ``rango/like_category/?category_id=XXX``
	- Instead of returning a HTML page have this view will return the new total number of likes for that category.
* Now in "rango-ajax.js" add the JQuery code to perform the AJAX GET request.
	- If the request is successful, then update the ``#like_count`` element, and hide the like button.


Updating Category Template
..........................
To prepare the template we will need to add in the "Like" button with ``id="like"`` and create a ``<div>`` to display the number of likes ``{{% category.likes %}}``. To do this, add the following ``<div>`` to the *category.html* template:

.. code-block:: html
	
	<p>
	
	<strong id="like_count">{{ category.likes }}</strong> people like this category
	
	{% if user.is_authenticated %}
		<button id="likes" data-catid="{{category.id}}" class="btn btn-primary" type="button">
		<span class="glyphicon glyphicon-thumbs-up"></span> 
		Like
		</button>
	{% endif %}
	
	</p>


Create a Like Category View
...........................
Create a view called, ``like_category`` in ``rango/views.py`` which will examine the request and pick out the category_id and then increment the number of likes for that category. 

.. code-block:: python
	
	from django.contrib.auth.decorators import login_required

	@login_required
	def like_category(request):
	   
	    cat_id = None
	    if request.method == 'GET':
	        cat_id = request.GET['category_id']

	    likes = 0
	    if cat_id:
	        cat = Category.objects.get(id=int(cat_id))
	        if cat:
		    likes = cat.likes + 1
	            cat.likes =  likes 
	            cat.save()
		
	    return HttpResponse(likes)

On examining the code, you will see that we are only allowing authenticated users to denote that they like a category. The view assumes that a variable ``category_id`` has been passed through via a GET so that the we can identify the category to update. In this view, we could also track and record that a particular user has "liked" this category if we wanted - but we are keeping it simple to focus on the AJAX mechanics.

Don't forget to add in the URL mapping, into ``rango/urls.py``. Update the ``urlpatterns`` by adding in:

.. code-block:: python
	
	url(r'^like_category/$', views.like_category, name='like_category'),


Making the AJAX request
.......................
Now in "rango-ajax.js" you will need to add some JQuery code to perform an AJAX GET request. Add in the following code:

.. code-block:: javascript
	
	    $('#likes').click(function(){
	        var catid;
	        catid = $(this).attr("data-catid");
	        $.get('/rango/like_category/', {category_id: catid}, function(data){
	                   $('#like_count').html(data);
	                   $('#likes').hide();
	        });
	    });

This piece of JQuery/Javascript will add an event handler to the element with id ``#likes``, i.e. the button. When clicked, it will extract the category id from the button element, and then make an AJAX GET request which will make a call to ``/rango/like_category/`` encoding the ``category_id`` in the request. If the request is successful, then the HTML element with id like_count (i.e. the <strong> ) is updated with the data returned by the request, and the HTML element with id likes (i.e. the <button>) is hidden.

There is a lot going on here and getting the mechanics right when constructing pages with AJAX can be a bit tricky. Essentially here, when the button is clicked an AJAX request is made, given our url mapping, this invokes the ``like_category`` view which updates the category and returns the new number of likes. When the AJAX request receives the response it updates parts of the page, i.e. the text and the button. The ``#likes`` button is hidden.



Adding Inline Category Suggestions
----------------------------------
It would be really neat if we could provide a fast way for users to find a category, rather than browsing through a long list. To do this we can create a suggestion component which lets users type in a letter or part of a word, and then the system responds by providing a list of suggested categories, that the user can then select from. As the user types a series of requests will be made to the server to fetch the suggested categories relevant to what the user has entered. 


Workflow
........
To do this you will need to do the following.

* Create a parameterised function called ``get_category_list(max_results=0, starts_with='')`` that returns all the categories starting with ``starts_with`` if ``max_results=0`` otherwise it returns up to ``max_results`` categories.
	- The function returns a list of category objects annotated with the encoded category denoted by the attribute, ``url``
* Create a view called *suggest_category* which will examine the request and pick out the category query string.
	- Assume that a GET request is made and attempt to get the *query* attribute.
	- If the query string is not empty, ask the Category model to get the top 8 categories that start with the query string.
	- The list of category objects will then be combined into a piece of HTML via template. 
* Instead of creating a template called ``suggestions.html`` re-use the ``cats.html`` as it will be displaying data of the same type (i.e. categories).
* To let the client ask for this data, you will need to create a URL mapping; lets call it *category_suggest*

With the mapping, view, and template for this view in place, you will need to update the ``base.html`` template and add in some javascript so that the categories can be displayed as the user types.

* In the ``base.html`` template modify the sidebar block so that a div with an id="cats" encapsulates the categories being presented. The JQuery/AJAX will update this element.
	- Above this <div> add an input box for a user to enter the letters of a category, i.e.:

		``<input  class="input-medium search-query" type="text" name="suggestion" value="" id="suggestion" />``
	
* With these elements added into the templates, you can add in some JQuery to update the categories list as the user types.
	- Associate an on keypress event handler to the *input* with ``id="suggestion"``
	- ``$('#suggestion').keyup(function(){ ... })``
	- On keyup, issue an ajax call to retrieve the updated categories list
	- Then use the JQuery ``.get()`` function i.e. ``$(this).get( ... )``
	- If the call is successful, replace the content of the <div> with id="cats" with the data received.
	- Here you can use the JQuery ``.html()`` function i.e. ``$('#cats').html( data )``

Parameterise the Get Category List Function
...........................................
In this helper function we use a filter to find all the categories that start with the string supplied. The filter we use will be ``istartwith``, this will make sure that it doesn't matter whether we use upper-case or lower-case letters. If it on the other hand was important to take into account whether letters was upper-case or not you would use ``startswith`` instead. 

.. code-block:: python

	def get_category_list(max_results=0, starts_with=''):
		cat_list = []
		if starts_with:
			cat_list = Category.objects.filter(name__istartswith=starts_with)
		
		if cat_list and max_results > 0:
			if cat_list.count() > max_results:
				cat_list = cat_list[:max_results]
			
		return cat_list

Create a Suggest Category View
..............................
Using the ``get_category_list`` function we can now create a view that returns the top 8 matching results as follows: 

.. code-block:: python
	
	def suggest_category(request):
		
		cat_list = []
		starts_with = ''
		if request.method == 'GET':
			starts_with = request.GET['suggestion']
		
		cat_list = get_category_list(8, starts_with)
			
		return render(request, 'rango/cats.html', {'cat_list': cat_list })

Note here we are re-using the ``rango/cats.html`` template :-). 

Map View to URL
...............
Add the following code to ``urlpatterns`` in ``rango/urls.py``:

.. code-block:: python

	url(r'^suggest_category/$', views.suggest_category, name='suggest_category'),

Update Base Template
....................
In the base template in the sidebar div add in the following HTML code:

.. code-block:: html

	
		<ul class="nav nav-list">
			<li class="nav-header">Find a Category</li>
			<form>
			<label></label>
			<li><input  class="search-query span10" type="text" name="suggestion" value="" id="suggestion" /></li>
			</form>
		</ul>
	
		<div id="cats">
		</div>	

Here we have added in an input box with ``id="suggestion"`` and div with ``id="cats"`` in which we will display the response. We don't need to add a button as we will be adding an event handler on keyup to the input box which will send the suggestion request.

Add AJAX to Request Suggestions
...............................
Add the following JQuery code to the ``js/rango-ajax.js``:

.. code-block:: javascript
	
	$('#suggestion').keyup(function(){
		var query;
		query = $(this).val();
		$.get('/rango/suggest_category/', {suggestion: query}, function(data){
                 $('#cats').html(data);
		});
	});

Here, we attached an event handler to the HTML input element with ``id="suggestion"`` to trigger when a keyup event occurs. When it does the contents of the input box is obtained and placed into the ``query`` variable. Then a AJAX GET request is made calling ``/rango/category_suggest/`` with the ``query`` as the parameter. On success, the HTML element with id="cats" i.e. the div, is updated with the category list html.

Exercises
---------
To let registered users quickly and easily add a Page to the Category put an "Add" button next to each search result.

* Update the ``category.html`` template:
	- Add a mini-button next to each search result (if the user is authenticated), garnish the button with the title and url data, so that the JQuery can pick it out.
	- Put a <div> with ``id="page"`` around the pages in the category so that it can be updated when pages are added.
	- Remove that link to add button, if you like. 
* Create a view auto_add_page that accepts a parameterised GET request (title, url, catid) and adds it to the category
* Map an url to the view ``url(r'^auto_add_page/$', views.auto_add_page, name='auto_add_page'),``
* Add an event handler to the button using JQuery - when added hide the button. The response could also update the pages listed on the category page, too.


Hints
.....
HTML Template code: 

.. code-block:: html
	
	{% if user.is_authenticated %}
		<button data-catid="{{category.id}}" data-title="{{ result.title }}" data-url="{{ result.link }}" class="rango-add btn btn-mini btn-info" type="button">Add</button>
	{% endif %}

JQuery code:

.. code-block:: javascript

	$('.rango-add').click(function(){
	    var catid = $(this).attr("data-catid");
		var url = $(this).attr("data-url");
        	var title = $(this).attr("data-title");
        	var me = $(this)
	    	$.get('/rango/auto_add_page/', {category_id: catid, url: url, title: title}, function(data){
	                   	$('#pages').html(data);
	                   	me.hide();
	               		});
	    				});

Note here we are assigned the event handler to all the buttons with class ``rango-add``.

View code:

.. code-block:: python
	
	@login_required
	def auto_add_page(request):
	    cat_id = None
	    url = None
	    title = None
	    context_dict = {}
	    if request.method == 'GET':
	        cat_id = request.GET['category_id']
	        url = request.GET['url']
	        title = request.GET['title']
	        if cat_id:
	            category = Category.objects.get(id=int(cat_id))
	            p = Page.objects.get_or_create(category=category, title=title, url=url)

	            pages = Page.objects.filter(category=category).order_by('-views')

	            # Adds our results list to the template context under name pages.
	            context_dict['pages'] = pages

	    return render(request, 'rango/page_list.html', context_dict)
	

