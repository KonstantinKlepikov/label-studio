---
type: playground
order: 201
---

<div style="padding-left: 60px; max-width: 700px; margin: 0 auto; margin-bottom: 2em;">
    <h1>Playground</h1>
    Start typing in the config, and you can quickly preview the labeling interface. At the bottom of the page, you have live serialization updates of what Label Studio expects as an input and what it gives you as a result of your labeling work.
</div>


{% raw %}

<style scoped>
 .content {
     max-width: none !important;
     padding-right: 50px;
     margin-left: 0px !important;
 }

 .validation {
     margin-top: 1em;
     margin-left: 1em;
     color: red;
 }
 
 iframe {
     border: 0px;
 }

 #render-editor {
     width: 100%;
 }

 #editor-wrap {
     background-color: rgb(252,252,252);
     padding: 5px;
     display: none;
 }

 .preview {
     padding: 5px;
     overflow: auto;
 }
 
.editor-row {
    display: flex; 
    margin-bottom: 1em; 
    width: 100%;
}

 .data-row {
    display: flex;
 }
 
.preview-col {
    width: 60%; 
    flex: 1; 
    background: rgb(252,252,252);
 }

 .editor-area {
 border: 1px solid #f48a4259;
 }
 
 .config-col {
    color: rgba(0,0,0,.6); 
    background: rgb(252,252,252); 
    margin-right: 2em; 
    width: 400px; 
    
 }

 .input-col {
 width: 49%;
 margin-right: 2%;
 }

 .output-col {
    width: 49%;
 }
 
@media screen and (max-width: 900px) {
.editor-row {
    flex-direction: column;
 }

 .data-row {
    flex-direction: column;
 }

.preview-col {
    width: 100%;
}

.config-col {
    widht: 100%;
 }
 
 .input-col, .output-col { width: 100%; }
 
}
 
</style>

<div>
  <div class="editor-row">
    <div class="config-col">
      <h3>Config</h3>
      <div class="editor-area">
      <!-- Textarea -->
      <textarea name="label_config" cols="40" rows="10" class="project-form htx-html-editor"
                id="id_label_config"></textarea>
      <div class="validation"></div>
      </div>
    </div>
    <div class="preview-col">
      <h3>Preview</h3>
      <div id="editor-wrap"></div>
      <pre class="preview" id="preload-editor">...</pre>
    </div>
  </div>
</div>
<!--  & Preview in two cols -->
<div class="data-row">
  <div class="input-col">
    <h3>Input Preview</h3>
    <div>
      <pre class="preview" id="upload-data-example">...</pre>
    </div>
  </div>
  <div class="output-col">
    <h3>Output Preview</h3>
    <div class="ui positive message">
      <pre class="preview" id="data-results">...</pre>
    </div>
  </div>
</div>
</div>

<script>
 function debounce(func, wait, immediate) {
     let timeout;

     return function () {
         const context = this, args = arguments;
         const later = () => {
             timeout = null;
             if (!immediate) func.apply(context, args);
         };
         const callNow = immediate && !timeout;

         clearTimeout(timeout);
         timeout = setTimeout(later, wait);
         if (callNow) func.apply(context, args);
     };
 }

 $(function () {
     // serialize editor output by timer
     setInterval(function () {
         if (typeof Htx !== 'undefined') {
             $('#data-results').text(JSON.stringify(Htx.completionStore.selected.serializeCompletion(), null, 4));
             $('#data-results').parent().show();
         }
     }, 500);

     var host = "https://go.heartex.net";
     var url_string = window.location.href
     var url = new URL(url_string);
     
     // Label code mirror
     let labelEditor = CodeMirror.fromTextArea(document.getElementById('id_label_config'), {
         lineNumbers: true,
         mode: "text/html",         
     });

     var _c = url.searchParams.get("config");
     if (_c && _c.length > 0) {
         console.log("wtf");
         var config = url.searchParams.get("config");
         config = config.replace(/[<][b][r][>]/gm, "\n");
         labelEditor.setValue(config);
         validate_config();
     } else {
         labelEditor.setValue("<View>\n  \n</View>");
     }
     
     // refresh for proper line numbers drawing
     labelEditor.refresh();
     // add validation
     labelEditor.on('change', debounce(function (editor) {
         validate_config(editor);
     }, 500));

     window.labelEditor = labelEditor;

     function validate_name() {
         let name = $('#id_title').val();
         validation_message('', 0);         
         return 0;
     }

     function validation_message(msg, status) {
         let o = $('.validation');
         o.text(msg);

         if (status === -1) {
             o.removeClass('hidden');
             o.addClass('visible');
         }
         if (status === 0) {
             o.removeClass('visible');
             o.addClass('hidden');
         }
     }

     // storage of validation results
     // let is_collection_ok = false;
     let is_label_ok = false;

     function editor_iframe(res) {
         // generate new iframe
         let iframe = $('<iframe><iframe>');
         iframe.className="editor-preview";
         // add iframe to wrapper div
         $('#editor-wrap').append(iframe);
         $('#editor-wrap').fadeIn();
         
         iframe.on('load', function () {
             // remove old iframe
             $('#render-editor').hide();
             $('#render-editor').remove();
             // assign id to new iframe
             iframe.attr('id', 'render-editor');
             iframe.show();
             // set height for iframe
             let obj = document.getElementById('render-editor');
             obj.style.height = (obj.contentWindow.document.body.scrollHeight + 30) + 'px';
             // hide "..."
             $('#preload-editor').hide();
         });

         // load new data into iframe
         iframe.attr('srcdoc', res);
     }

     function show_render_editor(editor) {
         let config = labelEditor.getValue();
         $.ajax({
             url: host + '/demo/render-editor?full_editor=t&config=' + encodeURIComponent(config),
             method: 'GET',
             success: editor_iframe,
             error: function() {
                 $('#preload-editor').show();
             }
         })
     }
     
     // send request to server with configs to validate
     function validate_config(editor) {

         // get current scheme type from current editor
         let url = host + '/api/projects/validate/';
         var val = labelEditor.getValue();

         if (!val.length)
             return;

         // label config validation
         $.ajax({
             url: url,
             method: 'POST',
             data: {label_config: val},
             success: function (res) {
                 is_label_ok = true;
                 validation_message('', 0);
                 $('#render-editor').show();
                 show_render_editor(editor);
                 // check_submit_button();
             },
             error: function (res) {
                 is_label_ok = false;
                 validation_message(res.responseJSON['label_config'][0], -1);
                 $('#render-editor').hide();
                 // check_submit_button();
             }
         });

         // load sample task
         $.get({
             url: host + '/business/projects/upload-example/',
             data: {label_config: val}
         })
          .fail(o => {
              $('#upload-data-example').text('...')
          })
          .done(o => {
              $('#upload-data-example').text(JSON.stringify(JSON.parse(o), null, 4))
          })
     }

 });
</script>


{% endraw %}
