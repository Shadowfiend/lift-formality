# Formality

Formality is a library that provides better, cleaner, more palatable
form handling for Lift. Form handling without the pain, if you will.

The goal of creating this library was to eliminate as much repetition as
possible while wielding the full power of Lift's stateful form mechanism
and its CSS selector transforms, as well as embracing Lift's own
error-reporting mechanisms.

## Status

Formality is still in an early state. While there are tests ensuring
everything works, use in production is likely to uncover some
undiscovered bugs. Those bugs will be crushed as quickly as possible
once reported on Github.

## Including

You can include Formality directly as a dependency in your `build.sbt`,
suffixing the major and minor Lift version you are using (3.0 thru 3.3 are
currently supported).

For example to include the build for 3.3, you would write:

```
libraryDependencies ++= Seq(
  "com.hacklanta" %% "lift-formality_3.3" % "1.2.0"
)
```

And for 3.2 you would write:

```
libraryDependencies ++= Seq(
  "com.hacklanta" %% "lift-formality_3.2" % "1.2.0"
)
```

And soforth.

## Usage

Formality provides a DSL for binding forms already defined in the HTML.
It leaves as much of the markup as possible to the markup itself, and
layers on only the necessary functionality from Lift's side. Amongst
other things, this means that the `type` of a field in the HTML is left
to the markup.

A basic form snippet would look like this:

```scala
  import com.hacklanta.formality.Formality._

  val registrationForm =
    form withFields(
      field[String](".name"),
      field[String](".phone-number"),
      field[Int](".age"),
      checkboxField("#terms-and-conditions")
    ) onSuccess { (name, phoneNumber, age, termsAndConditions) =>
      // Assuming a case class User(name: String, age: Int, phoneNumber: String, termsAndConditions: Boolean).
      User(name, age, phoneNumber, termsAndConditions).save
    }

  "form" #> registrationForm.binder
```

The corresponding HTML5 markup could be:

```html
<form data-lift="Registration.form">
  <input class="name">
  <input class="phone-number" placeholder="(XXX) XXX-XXXX">
  <input class="age" type="number">

  <label for="terms-and-conditions>
    <input type="checkbox" id="terms-and-conditions">
    I agree to the <a href="/terms">Terms and Conditions</a>
  </label>
</form>
```

This is a very basic registration form with no validations or event
callbacks. The key here is that our success handler gets a set of
properly-typed values. It is only called if all of the entries can
properly be deserialized into their requested types (for example,
putting in "hello" for the age would mean the success callback would not
run).

Behind the scenes, Formality tries to deserialize the values. If
the deserialization fails, `S.error` is automatically set for the
appropriate field. You can then customize the behavior of `S.error` as
you desire (see [LiftRules](http://liftweb.net/api/26/api/net/liftweb/http/LiftRules.html)
for more, specifically the `noticesToJsCmd` property).

### Validations

Formality can also do validation:

```scala
  import com.hacklanta.formality.Formality._
  import com.hacklanta.formality.Html5Validations._

  val nameField = field[String](".name") ? notEmpty
  val phoneNumberField = field[String](".phone-number")
  val ageField = field[Int](".age") ? inRange(15, 120)
  val termsField =
    checkboxField("#terms-and-conditions") ? { incoming: Boolean =>
      if (incoming) {
        Empty
      } else {
        Full("You must agree to the terms and conditions.")
      }
    }

  val registrationForm =
    form withFields(
      nameField,
      phoneNumberField,
      ageField,
      termsField
    ) onSuccess {
      // same as above
    }
```

Now we've taken it a step further. Validations can be simple
functions. These functions take the deserialized value (i.e., the value
after it has been typed properly) and return a `Full` `Box` with an
error message when there is an error, or an `Empty` `Box` if the value
is valid. In cases where a validation fails (by providing an error
message), that error message is associated with the appropriate field by
calling `S.error`.

Validations can also be more than simple functions. The built-in
validations, like `inRange` and `notEmpty`, extend the `Validation`
trait. These `Validation`s are still functions that work as specified
above (i.e., they have an `apply` method that behaves as we specified
above). However, they also have a `binder` method that returns a
`CssSel`. These can be used to provide client-side attributes to also
enforce the validation client-side. Client-side validations have
the advantage of being able to run faster and without taxing the
server. `Validation`s allow you to specify client-side validations and
pair them with server-side validations to ensure that valid data is what
makes it into your database.

The two built-ins used above are from the `Html5Validations` object,
which does client-side validation using HTML5 attributes. In particular,
the `notEmpty` validation applies a `required` attribute to the input,
and the `inRange` validation applies `min` and `max` attributes to the
input.

Also in progress is an object, `ParsleyValidations`, meant to support
the set of validations that are supported by
[Parsley.js](http://parsleyjs.org). These will have matching server-side
validation implementations.

In certain very specific cases, it can be useful to run a validation even
if the value in question isn't actually submitted in the form. Typically
this is used for a check that a required field was submitted, particularly
when it comes from a checkbox or radio button. Either way, in these cases
a validation can take a `Box[T]` instead of `T`. It will receive an `Empty`
if the field is not submitted, and a `Full` with the deserialized value if
the field is submitted. Lift-formality's own `notBlank`/`notEmpty` validators
will correctly handle unsubmitted fields.

### Event handling

Formality can also add server-side event handlers to form inputs:

```scala
  import com.hacklanta.formality.Formality._
  import com.hacklanta.formality.Html5Validations._

  val nameField = field[String](".name") ? notEmpty
  val phoneNumberField = field[String](".phone-number")
  val ageField =
    field[Int](".age") ?
      inRange(15, 120) ->
      on("change", { incoming: Int =>
        if (incoming < 15) {
          Run("$('#terms-and-conditions').attr('disabled', 'disabled')")
        }
      })
  val termsField =
    checkboxField("#terms-and-conditions") ? { incoming: Boolean =>
      if (incoming) {
        Empty
      } else {
        Full("You must agree to the terms and conditions.")
      }
    }

  val registrationForm =
    form withFields(
      nameField,
      phoneNumberField,
      ageField,
      termsField
    ) onSuccess {
        // same as above
    }
```

`-> on(<event>, <handler)` is the syntax used to add an event handler to
a field. The `event` is whatever the event name will be on the client, as a
`String`, and the `handler` takes in the deserialized incoming value and
does something with it. It is expected to return a `JsCmd`, though keep
in mind that there is an implicit conversion from `Unit` to `JsCmd`
when necessary. Above, on change, we disable the terms and conditions
checkbox to ensure the user cannot select it if they are below
15[*](#client-note)<a name="client-note-return"></a>.

<a name="client-node"></a>* - Obviously we could do this on the client as
well, but this is merely an example :) [↩](#client-note-return)

### File Uploads

A complicating factor is when your form includes uploaded files.

They will work perfectly if your form is not ajaxified, but submitting files as part of an ajax upload takes a little more effort.  A classic approach is insert an iframe and submit to that, thus preventing your page from reloading.

This can be implemented clientside with the inclusion of a script along the lines of:
(Adapted from [this example](https://github.com/Shadowfiend/lift-ajax-file-upload-example/blob/master/src/main/webapp/static/js/fileUpload.js]))

```javascript
$(function() {
    function submitFormToIframe($form, target) {
        $form
            .attr('target', target)
            .removeAttr('onsubmit')
            .removeAttr('action')
            .removeAttr('onclick')
            .attr('action', '/ajax')
            .attr('method', 'post')
            .attr('enctype', 'multipart/form-data' )
            .attr('encoding', 'multipart/form-data')
            .find('input:submit,button[type=submit]')
            .end()
            .append($('<input type="hidden" name="' +
                      $form.find('input:submit').attr('name') +
                      '" value="_" />'))
            .after(
                // do not use attr() to set name. IE7 will hate this: http://stackoverflow.com/questions/2105815/weird-behaviour-of-iframe-name-attribute-set-by-jquery-in-ie
                $('<iframe id="' + target + '" name="' + target + '" />')
                    .addClass('form-target')
                    .css('display','none')
            );
    }

    submitFormToIframe($('form:has(input[type=file])'), 'fileUploadExampleForm');
});
```
You may also wish to treat your uploaded files as optional.  By default, a `fileUploadField` will fail validation if not provided. You can make the field(s) optional through a conversion into Option:

```scala
fieldGroup
            .withFields(
              fileUploadField("#anInputOfTypeFile")
            )
            .withBoxedConverter { optional: Box[FileParamHolder] =>
              optional match {
                case Full(fph) => Full(Some(fph))
                case _         => Full(None)
              }
            }
```

### Operator Allergies

In case you have an aversion to using operators, the `?` operator that
adds a validation to a field is originally named `validatingWith`, so
you can use that instead. Additionally, the `->` operator that adds
an event handler is originally named `handlingEvent`.

### Failure Handling

Formality's default behavior is to send down deserialization and
validation errors using `S.error`. However, you may want to take
additional action when dealing with failures. Or, you may want to
ignore `S.error` in favor of your own error handling strategy. To
do this, you can add a failure handler to the form:

```scala
  val registrationForm =
    form withFields(
      nameField,
      phoneNumberField,
      ageField,
      termsField
    ) onSuccess {
        // same as above
    } onFailure { failures =>
      failures.foreach {
        case ParamFailure(message, _, _, validationErrors) =>
          logger.error("Got " + message + " with validation errors: " + validationErrors)
        case Failure(message, _, _) =>
          logger.error("Got " + message)
        case _ =>
      }
    }
```

Note that the basic `Failure` message when validations fail simply says
"<field> failed validations." However, in such cases, the failure is
a `ParamFailure` whose parameter is a list of `String`s representing the
validation errors for that field.

### Field Groups

Formality provides an abstraction above fields, called field groups. This
lets you group fields into nested groups. By default, this looks like this:

```scala
  val registrationForm =
    form withFields(
      fieldGroup.withFields(
        nameField,
        phoneNumberField,
        ageField
      ),
      termsField
    ) onSuccess {
      case ((name :+: phoneNumber :+: age :+: HNil), terms) =>
        // Assuming a case class User(name: String, age: Int, phoneNumber: String) this time.
        if (terms) {
          User(name, phoneNumber, age).save
        }
      User(name, age, phoneNumber, termsAndConditions).save
    }
```

But field groups also have a very important property: they can be converted
into a different type using its `as` function. In particular, a field group can
be a placeholder for a type that involves its combined fields. Let's redo the
above:

```scala
  val registrationForm =
    form withFields(
      fieldGroup.withFields(
        nameField,
        phoneNumberField,
        ageField
      ).as(User.apply _),
      termsField
    ) onSuccess { (user, terms) =>
      if (terms) {
        user.save
      }
    }
```

Our success handler now gets a full-on User object instead of the decomposed
fields, and we could use the `User` constructor directly.

`as` takes a function that takes the parameters from the group and produces a
type T. However, sometimes we want to be able to fail at the conversion step.
In those cases, we can use `withConverter` instead, which allows us to return a
`Box[T]`, where a non-`Full` will cause a failed form submission:

```scala
  val registrationForm =
    form withFields(
      fieldGroup.withFields(
        nameField,
        phoneNumberField,
        ageField
      ).withConverter(User.createUnique _),
      termsField
    ) onSuccess { (user, terms) =>
      if (terms) {
        user.save
      }
    }
```

Above, `createUnique` can be considered as a version of `User` that creates the
new `User` object and returns a `Full` unless, say, the name is not unique. In that case it returns a `Failure`.

Note that failures at the group conversion level don't trigger `S.error`, since
there's no concrete field to associate the failure with. You can do your own
handling of the failure in the `onFailure` handler.

#### Field groups with boxed converters

Field groups can also use a converter that takes the boxed values of the
fields that it is made up of. This can be used, for example, to make choices
between two fields that supply the same data. Let's take as an example an
upload form that accepts a document in PDF format or a pasted text file:

```scala
  val fileNameField = field[String](".file-name")
  val pastedTextField = field[String](".pasted-text")
  val uploadedFileField = fileUploadField(".pasted-text")

  val uploadForm =
    form withFields(
      fileNameField,
      fieldGroup.withFields(
        pastedTextField,
        uploadedFileField
      ).withBoxedConverter { (text: Box[String], file: Box[FileParamHolder]) =>
        text.map(_.getBytes("UTF-8")) or file.data
      }
    ).onSuccess { (fileName: String, fileData: Array[Data]))
      DbFile.create(fileName, fileData)
    }
```

Above, we have a form with a field for the file's name, and then a group that
encloses the pasted text and the file upload. The converter receives both as
boxed values, which will be `Empty` if the user didn't specify anything for the
fields in question. The converter then uses the pasted text if it was specified,
or if not chooses the file data. The success handler then only sees two parameters:
the files name, and the final data array that represents its contents, which we've
resolved in the field group converter.

#### Scoping field groups

You can also add a scope to a field group. This is a CSS selector that ensures
the fields in the field group are bound only if they're found within elements
that match that selector:

```scala
  val registrationForm =
    form withFields(
      fieldGroup(".user").withFields(
        nameField,
        phoneNumberField,
        ageField
      ).withConverter(User.createUnique _),
      termsField
    ) onSuccess ...
```

With the above code, the markup that we introduced at the beginning would no
longer successfully bind the user part of the registration form. Instead, we
would need something like this:

```html
<form data-lift="Registration.form">
  <fieldset class="user">
    <legend>User Information</legend>

    <input class="name">
    <input class="phone-number" placeholder="(XXX) XXX-XXXX">
    <input class="age" type="number">
  </fieldset>

  <label for="terms-and-conditions>
    <input type="checkbox" id="terms-and-conditions">
    I agree to the <a href="/terms">Terms and Conditions</a>
  </label>
</form>
```

## License

`lift-formality` is provided under the terms of the MIT License. No warranties
are made, express or implied.  See the `LICENSE` file in this same directory.

# Author/Contributors

`lift-formality` is copyright [Hacklanta](http://hacklanta.com), and was
originally conceived and written by [Antonio Salazar
Cardozo](http://github.com/Shadowfiend).

You can find our writings on the [Hacklanta blog](http://hacklanta.com/blog).
