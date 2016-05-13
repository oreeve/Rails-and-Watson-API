# Rails-and-Watson-API

The Watson Document Conversion API allows you to input a Word, PDF, or HTML document and convert it to plain text, HTML, or JSON format. In my app, I wanted to convert an uploaded PDF into JSON. Because the Watson API has no Ruby documentation, and Ruby doesn't natively support uploading a file through an HTTP request, I used the [faraday](https://github.com/lostisland/faraday) gem to format the HTTP request. I also used [carrierwave](https://github.com/carrierwaveuploader/carrierwave) for file uploading.

In my database I have a model called `Assignment` which has numerous properties, one of which is the property `file` which is a string. 


When creating a new assignment on the `/assignments/new` page, the file is
saved via the following form field:

```ruby
<div class="file">
  <%= f.label :file, "File (PDF Only)" %>
  <%= f.file_field :file %>
</div>
```
The Assignments controller had the necessary `new` and `create` methods to save
new assignments to the database, which I won't be covering here.

My converted document would be displayed on the Assignment "show" page
(`/assignments/:id`). This is where the call to the Watson API would take
place.

In the Assignments controller, I had the following code:

```ruby
app/controllers/assignments_controller.rb

def show
  @assignment = Assignment.find(params[:id])
  # This makes the API request with the file as an argument
  @document = WatsonApi.new(@assignment.file)
end
```

The `WatsonApi` wrapper class would take care of sending the request to the
API.

I used the [faraday](https://github.com/lostisland/faraday) gem to format the
HTTP request. The `WatsonApi` model contained the following:

```ruby
app/models/watson_api.rb

require 'json'
require 'open-uri' #For opening the file when not stored locally
require 'faraday'

class WatsonApi
  attr_reader :data

  # Methods called on the Assignment show page when you call WatsonApi.new

  def initialize(doc) #doc argument is the @assignment.file
    if Rails.env.production?
      # In production, the file is a url from an S3 bucket
      @doc = doc.url
    else
      # In development, it's a local storage path
      @doc = doc.path
    end
    # This calls the get_doc method whenever we make a WatsonApi.new request
    # from the show page and sets the response data equal to the @data variable
    @data = get_doc
  end

  def questions
    # This iterates through the returned JSON and pulls out the desired data,
    # in this case the questions from the Assignment.
    @data.values[3].map { |item| [item["title"], item["content"][0]["text"]] }
  end

  private

  def get_doc

    # Connecting to the API with authorizations

    base_url = "https://gateway.watsonplatform.net"
    version = "2015-12-15"
    path = "/document-conversion/api/v1/convert_document?version=#{version}"
    basic_auth_username = ENV["WATSON_USERNAME"]
    basic_auth_password = ENV["WATSON_PASSWORD"]
    # Env variables in your .env file, given to you by signing up with Watson

    conn = Faraday.new(url: base_url) do |faraday|
      faraday.request :multipart #multipart indicates that a file is being uploaded
      faraday.request :url_encoded
      faraday.adapter :net_http
    end

    conn.basic_auth(basic_auth_username, basic_auth_password)

    # Setting the payload configuration and adding your file to the request

    payload = {}
    if Rails.env.production?
      payload = {
        config: Faraday::UploadIO.new("#{::Rails.root}/lib/WatsonApi/config.json", "application/json"),
          # config tells Watson what you want your document converted to (JSON/HTML/PLAIN TEXT)
          # This is in a separate file config.json (see below)
        file: Faraday::UploadIO.new(open(@doc), "application/pdf")
          # In production, calling open on the @doc url with open-uri allows us
          # to open the url as if it were a file
      }
    else
      payload = {
        config: Faraday::UploadIO.new("#{::Rails.root}/lib/WatsonApi/config.json", "application/json"),
        file: Faraday::UploadIO.new(@doc, "application/pdf")
      	# This is the file and type to be uploaded. In development, @doc is a
      	# local file path so it can be sent without "opening" it first
      }
    end

    response = conn.post(path, payload)
    JSON.parse(response.body)
    # The parsed response will be returned and therefore
    # set as @data in the initializer
  end
end
```

The document conversion API
[docs](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/document-conversion/api/v1/#convert-document)
have a lot more information about configuring your request. For configuring the
file type you want Watson to return (in this case, JSON), the
`conversion_target` can be set to "answer_units", "normalized_html", or
"normalized_text".

```ruby
lib/WatsonApi/config.json

{ "conversion_target" : "ANSWER_UNITS" }
```


Because I was using an `Assignment` model to display assigned work to students,
my method to parse the returned data was called `questions`. I use this method,
seen above in the `WatsonApi` wrapper, on the assignment's show page to display
the converted document:

```html
app/views/assignments/show.html.erb

<% @document.questions.each do |item| %>
  <li>
    <h4><%= item[0] %></h4>
    <p><%= item[1] %></p>
  </li>
<% end %>
```
Where `item[0]` was a header or question and `item[1]` was the text beneath it.
I recommend looking at your JSON response to determine how you want to retrieve
data from your converted document.

The PDF has now been converted to JSON and displayed on the page!

For more options and further detail about formatting your request, checkout the
faraday [docs](https://github.com/lostisland/faraday/blob/master/README.md).
You can also look into the Ruby
[OpenURI](http://ruby-doc.org/stdlib-2.1.0/libdoc/open-uri/rdoc/OpenURI.html)
wrapper and [JSON](http://ruby-doc.org/stdlib-2.0.0/libdoc/json/rdoc/JSON.html)
module.

