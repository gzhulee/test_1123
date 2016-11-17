- if diff.a_mode && diff.b_mode && diff.a_mode != diff.b_mode
  .alert.alert-warning
    = t("merge_request.file_access_change")
- if !diff.nil? and diff.present?
  - too_big = diff.diff.lines.count > 5000 || diff.diff.to_s =~ /^FIleToBig======, \d+, \d+/

  - if too_big
    .alert.alert-danger
      = t("merge_request.change_too_much")
  - else
    - comments_template = "<a href=\"javascript:;\" class=\"add-diff-note js-add-diff-note-button\" data-parallel=%%parallel%% data-commit-id=\"#{@commit.id}\" data-discussion-id=\"discussion-commit--%%line_code%%\" data-line-code=\"%%line_code%%\" data-noteable-type=\"Commit\" title=\"#{t('merge_request.add_a_note')}\"><i class=\"fa fa-comments\"></i></a>"
    - if @view != 'parallel'
      %table.commit-diff
        - rugged_each_diff_line(diff, index) do |line, type, line_code, line_new, line_old, start_num, end_num|
          %tr.line_holder{:id => "in_line_#{end_num}"}
            - if type == "match"
              %td.old_line
                = link_to "...", get_more_code_project_commit_path(@project,@commit, start_num:start_num, end_num:end_num, file_name:diff.old_path, view:'inline'), remote:true
              %td.new_line
                = link_to "...", get_more_code_project_commit_path(@project,@commit, start_num:start_num, end_num:end_num, file_name:diff.old_path, view:'inline'), remote:true
              %td.line_content.matched= raw diff_line_content(line)
            - else
              %td.old_line
                = link_to raw(type == "new" ? "&nbsp;" : line_old), "##{line_code}"
                - if @comments_allowed
                  = raw(comments_template.gsub('%%line_code%%', line_code))
              %td.new_line= link_to raw(type == "old" ? "&nbsp;" : line_new), "##{line_code}"
              %td.line_content{class: "noteable_line #{type}", "line_code" => line_code}= raw diff_line_content(line)
          - if @reply_allowed
            - comments = @line_notes.select { |n| n.line_code == line_code }
            - unless comments.empty?
              = render "notes/line_code_notes", notes: comments,  hidden: false
    - else
      %table.commit-diff.parallel
        - parallel_diff_parallelize(diff).each do |line|
          %tr.line_holder{:id => "parallel_line_#{line[:left][:end_num]}"}
            - if line[:left][:type] == "match"
              %td.old_line
                = link_to "...", get_more_code_project_commit_path(@project,@commit, start_num:line[:left][:start_num], end_num:line[:left][:end_num], file_name:diff.old_path, view:'parallel'), remote:true
              %td.line_content.matched= raw diff_line_content(line[:left][:text])
              %td.new_line
                = link_to "...", get_more_code_project_commit_path(@project,@commit, start_num:line[:left][:start_num], end_num:line[:left][:end_num], file_name:diff.old_path, view:'parallel'), remote:true
              %td.line_content.parallel.matched= raw diff_line_content(line[:right][:text])
            - else
              %td.old_line{class: "#{line[:left][:type]}"}
                = link_to raw(line[:left][:type] == "new" ? "&nbsp;" : line[:left][:number]), "##{line[:left][:line_code]}"
                - if @comments_allowed
                  = raw(comments_template.gsub('%%line_code%%', line[:left][:line_code]).gsub("%%parallel%%", "left"))
              %td.line_content.parallel{class: "noteable_line #{line[:left][:type]}", "line_code" => line[:left][:line_code]}= raw diff_line_content(line[:left][:text])
              %td.new_line{class: "#{line[:right][:type]}"}
                = link_to raw(line[:right][:type] == "old" ? "&nbsp;" : "#{line[:right][:number]}"), "##{line[:right][:line_code]}"
                - if @comments_allowed
                  = raw(comments_template.gsub('%%line_code%%', line[:right][:line_code]).gsub("%%parallel%%", "right"))
              %td.line_content.parallel{class: "noteable_line #{line[:right][:type]}", "line_code" => line[:right][:line_code]}= raw diff_line_content(line[:right][:text])
          - if @reply_allowed
            fdsafdsfsdfsdafdsfdsa
            - comments = {"left" => [], "right" => []}
            - leftComments = @line_notes.select { |n| n.line_code == line[:left][:line_code] }
            - rightComments = @line_notes.select { |n| n.line_code == line[:right][:line_code] && line[:left][:line_code] != line[:right][:line_code]}
            - unless leftComments.empty?
              - comments["left"] = leftComments
            - unless rightComments.empty?
              - comments["right"] = rightComments
            - unless leftComments.empty? && rightComments.empty?
              = render "notes/line_code_notes", notes: comments