---
title: "Markus Project: Preliminary Research on Uploading Scanned Exam Process"
datePublished: Mon Feb 10 2025 04:24:42 GMT+0000 (Coordinated Universal Time)
cuid: cm6yjuciy000c09juh87yf3su
slug: markus-project-preliminary-research-on-uploading-scanned-exam-process
tags: ruby, javascript, web-development, ruby-on-rails, apis, reactjs

---

In this article, we will examine the end-to-end process of uploading a scanned test on Markus. Markus allows instructors to upload scanned test papers in pdf format for grading. A detailed guide for creating scanned assignments can be found at the [Wiki](https://github.com/MarkUsProject/Wiki/blob/master/Instructor-Guide--Scanned-Exams.md)

### Tracing the Upload Scan Log

After creating a scanned assignment and an exam template, we can upload test papers in the “Upload Scan“ page by selecting a template and uploading a pdf file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739072784022/b77a2878-8958-40e3-872e-2d89c48ed761.png align="center")

After clicking the upload button, we can trace the log of uploading a scanned exam, splitting it into individual pages, and polling the backend for the job status. We will focus on a few key steps in the log:

1. Uploading the scanned exam:
    

> \[instructor\] Started PATCH "/csc108/courses/1/assignments/20/exam\_templates/split" for 172.18.0.1 at 2025-02-08 21:40:33 -0500 \[instructor\] Processing by ExamTemplatesController#split as JS \[instructor\] Parameters: { "authenticity\_token"=&gt;"\[FILTERED\]", "exam\_template\_id"=&gt;"2", "pdf\_to\_split"=&gt;#&lt;ActionDispatch::Http::UploadedFile:0x00007f13c8d35058 @tempfile=#[Tempfile:/tmp/RackMultipart20250208-112-tbe1pj.pdf](Tempfile:/tmp/RackMultipart20250208-112-tbe1pj.pdf), @content\_type="application/pdf", @original\_filename="TermTest1\_20249\_Solutions.pdf"&gt;, "on\_duplicate"=&gt;"overwrite", "commit"=&gt;"Upload", "course\_id"=&gt;"1", "assignment\_id"=&gt;"20" }

The frontend sends a PATCH request to the `/split` route of `ExamTemplatesController`, with the uploaded pdf file as parameter. The backend processes this request and initializes a `SplitPdfJob` to split the pdf according to sections defined by the template.

2. Initiating the background job:
    

> \[instructor\] \[ActiveJob\] Enqueued SplitPdfJob (Job ID: bab489c4-4e55-4966-949d-286950a99588) to Resque(DEFAULT\_QUEUE) with arguments: #&lt;GlobalID:0x00007f13f5a17038 @uri=#&lt;URI::GID gid://markus/ExamTemplate/2&gt;&gt;, "/tmp/RackMultipart20250208-112-tbe1pj.pdf", #&lt;GlobalID:0x00007f13f5a14b08 @uri=#&lt;URI::GID gid://markus/SplitPdfLog/16&gt;&gt;, "TermTest1\_20249\_Solutions.pdf", #&lt;GlobalID:0x00007f13f5a12e20 @uri=#&lt;URI::GID gid://markus/Instructor/1&gt;&gt;, "overwrite"

The `SplitPdfJob` is enqueued with various parameters.

3. Polling for job status:
    

> \[instructor\] Started GET "/csc108/job\_messages/bab489c4-4e55-4966-949d-286950a99588/get" for 172.18.0.1 at 2025-02-08 21:40:33 -0500
> 
> \[instructor\] Processing by JobMessagesController#get as JSON
> 
> \[instructor\] Parameters: {"job\_id"=&gt;"bab489c4-4e55-4966-949d-286950a99588"}

After enquereing the job, the fronted sends periodic GET requests to the `get` route of `JobMessagesController` to poll the job status. The job id is passed as a parameter.

4. Rendering the polling UI
    

> \[instructor\] Rendering exam\_templates/\_poll\_generate\_job.js.erb
> 
> \[instructor\] Rendered shared/\_poll\_job.js.erb (Duration: 0.3ms | GC: 0.0ms)
> 
> \[instructor\] Rendered exam\_templates/\_poll\_generate\_job.js.erb (Duration: 0.5ms | GC: 0.0ms)

The backend renders an HTML view that updates the frontend UI with the job status.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739073611643/c46dd857-a2dd-4b02-a0e7-acc2226bc937.png align="center")

5. Job completion
    

> \[instructor\] Completed 200 OK in 72ms (Views: 1.6ms | ActiveRecord: 7.3ms (17 queries, 3 cached) | GC: 7.3ms)

The backend completes the job and in the next step goes back the to main “Exam Template“ page.

Through this process, we can identify these few components to look into:

1. `ExamTemplatesController`: the controller that initializes the split pdf job.
    
2. `JobMessageController`: the controller the provides job status when polled.
    
3. `SplitPdfJob` and `GenerateJob`: the `ActiveJob` that performs the pdf processing
    
4. `_poll_job.js.erb`: the frontend component that performs the polling.
    
5. `_poll_generate_job.js.erb`: the component that initializes the polling process.
    

### Rails Active Job

Active Job is part of Rails framework to run time-consuming tasks in the background without blocking the main application and user request-response cycles. Scheduled active jobs are stored in a job queue and polled by Rails to process. Active jobs are located in `app/jobs` directory. Below is the definition of `SplitPdfJob` with implementation details omitted:

```ruby
class SplitPdfJob < ApplicationJob
  def self.on_complete_js(_status)
    'window.location.reload.bind(window.location)'
  end

  def self.show_status(status)
    I18n.t('poll_job.split_pdf_job', progress: status[:progress],
                                     total: status[:total],
                                     exam_name: status[:exam_name])
  end

  before_enqueue do |job|
    status.update(exam_name: "#{job.arguments[0].name} (#{job.arguments[3]})")
  end

  def perform(exam_template, _path, split_pdf_log, _original_filename = nil, _current_role = nil, on_duplicate = nil)
    # implementation omitted
  end

  # Save the pages into groups for this assignment
  def save_pages(exam_template, partial_exams, filename = nil, split_pdf_log = nil, on_duplicate = nil)
    # implementation omitted
  end

  # Determine a match using the parsed handwritten text and the user identifying field (+exam_template.cover_fields+).
  # If the parsing was successful, +parsed+ is a string parsed from the handwritten text: if
  # +exam_template.cover_fields+ is 'id_number', it is the result of attempting to parse numeric digits, and if
  # +exam_template.cover_fields+ is 'user_name', it is the result of attempting to parse alphanumeric characters.
  def match_student(parsed, exam_template)
    # implementation omitted
  end

  def group_name_for(exam_template, exam_num)
    "#{exam_template.name}_paper_#{exam_num}"
  end

  def get_num_groups_in_dir(dir)
    # implementation omitted
end
```

The `perform` method contains the code to be executed when the job is polled. In this case, it is the code that performs the pdf spliting. There are also other methods called at different stages of job lifecycle:

* [`before_enqueue`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-before_enqueue)
    
* [`around_enqueue`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-around_enqueue)
    
* [`after_enqueue`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-after_enqueue)
    
* [`before_perform`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-before_perform)
    
* [`around_perform`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-around_perform)
    
* [`after_perform`](https://api.rubyonrails.org/v8.0.1/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-after_perform)
    

A job is enquered using `perform_later` method, as found in `exam_template` model

```ruby
  def split_pdf(path, original_filename = nil, current_role = nil, on_duplicate = nil)
    basename = File.basename path, '.pdf'
    filename = original_filename.nil? ? basename : File.basename(original_filename)
    pdf = CombinePDF.load path
    num_pages = pdf.pages.length

    # creating an instance of split_pdf_log
    split_pdf_log = SplitPdfLog.create(
      exam_template: self,
      filename: filename,
      original_num_pages: num_pages,
      num_groups_in_complete: 0,
      num_groups_in_incomplete: 0,
      num_pages_qr_scan_error: 0,
      role: current_role
    )

    raw_dir = File.join(self.base_path, 'raw')
    FileUtils.mkdir_p raw_dir
    FileUtils.cp path, File.join(raw_dir, "raw_upload_#{split_pdf_log.id}.pdf")

    SplitPdfJob.perform_later(self, path, split_pdf_log, original_filename, current_role, on_duplicate)
  end
```

We are able to inspect specific jobs by job id in the job queue. The `job_message_controller` provides an endpoint for frontend to poll the job status of a given job. In our current implementation, the frontend repeated sends GET request to `jobMessageController` to check for the status of split pdf job, and updates the frontend when completed.

For our next task, we will research about an “interrupt based“ method for frontend to know job update instead of having the frontend doing repeated polling. We will explore an approach based on websocket.

```ruby
class JobMessagesController < ApplicationController
  before_action { authorize! }

  def get
    status = ActiveJob::Status.get(params[:job_id])
    if status.failed?
      flash_message(:error, t('poll_job.failed')) if status[:exception].blank? || status[:exception][:message].blank?
      session[:job_id] = nil
    elsif status.completed?
      status[:progress] = status[:total]
      completed_message = status[:job_class].completed_message(status)
      if completed_message.is_a?(Hash)
        flash_message(:success, **completed_message)
      else
        flash_message(:success, completed_message)
      end
      session[:job_id] = nil
    elsif status.read.empty?
      flash_message(:error, t('poll_job.failed'))
      session[:job_id] = nil
      render json: { code: '404', message: t('poll_job.not_enqueued') }, status: :not_found
      return
    end
    flash_job_messages(status)
    render json: status.read
  end

  private

  def flash_job_messages(status)
    flash_message(:error, status[:exception][:message]) if status[:exception].present?
    flash_message(:warning, status[:warning_message]) if status[:warning_message].present?
    current_status = status[:job_class]&.show_status(status)
    if current_status.nil? || session[:job_id].nil?
      hide_flash :notice
    else
      status.queued? ? flash_message(:notice, t('poll_job.queued')) : flash_message(:notice, current_status)
    end
  end

  protected

  def implicit_authorization_target
    OpenStruct.new policy_class: JobMessagePolicy
  end

  def parent_params
    []
  end

  # Job messages are allowed while switching courses
  def check_course_switch; end
end
```