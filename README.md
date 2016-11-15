Hello branch_op_ddd
# coding: utf-8

require 'gitlab/grit_diff'

class ApplicationController < ActionController::Base
  include AuthenticatedSystemSameDomain

  #Stored for SSO rediect back
  before_filter :store_current_url
  before_filter :synchronize_login_status_from_csdn_passport

  before_filter :require_login?
  before_filter :current_user
  before_filter :set_current_user_for_observers
  before_filter :add_abilities
  #before_filter :dev_tools if Rails.env == 'development'
  before_filter :default_headers
  before_filter :set_language

  # before_filter :check_visit_snippets

  #before_filter :notifications_count

  protect_from_forgery

  helper_method :abilities, :can?

  rescue_from Encoding::CompatibilityError do |exception|
    log_exception(exception)
    render_500
  end

  rescue_from ActiveRecord::RecordNotFound do |exception|
    log_exception(exception)
    render_404
  end

  render_drb_server_time_out
  rescue_from DRb::DRbConnError do |exception|
  rescue_from DRb::DRbTimeout do |exception|
    log_exception(exception)
    render_drb_server_time_out
  end

  protected

  def log_exception(exception)
    application_trace = ActionDispatch::ExceptionWrapper.new(env, exception).application_trace
    application_trace.map!{ |t| "  #{t}\n" }
    logger.error "\n#{exception.class.name} (#{exception.message}):\n#{application_trace.join}"
  end

  def check_bi_statistic_conf
    render_404 unless Gitlab::SystemConfigsUtil.bi_statistic_conf
  end

  def reject_blocked!
    if current_user && current_user.blocked
      sign_out current_user
      flash[:alert] = "Your account is blocked. Retry when an admin unblock it."
      redirect_to new_user_session_path
    end
  end

  # def after_sign_in_path_for resource
  #   if resource.is_a?(User) && resource.respond_to?(:blocked) && resource.blocked
  #     sign_out resource
  #     flash[:alert] = "Your account is blocked. Retry when an admin unblock it."
  #     new_user_session_path
  #   else
  #     super
  #   end
  # end

  def set_current_user_for_observers
    ProjectObserver.current_user = current_user
    MergeRequestObserver.current_user = current_user
    IssueObserver.current_user = current_user
    SnippetObserver.current_user = current_user
    SnippetStarObserver.current_user = current_user
    RecommendDocObserver.current_user = current_user
    KeyObserver.current_user = current_user
  end

  def abilities
    @abilities ||= Six.new
  end

  def can?(object, action, subject)
    abilities.allowed?(object, action, subject)
  end

  def project
    id = params[:project_id] || params[:id]
    @project = Project.find_with_namespace(id) unless id.nil?
    @project
    # project function just get the project record from db. and the leave the auth job to authorize_project.
    # Rencs 2016-11-8
    # if @project and can?(current_user, :read_project, @project) and !@project.blocked
    #   @project
    # elsif @project and  @project.audit_flag.to_i == 3 and !@project.blocked
    #   @project = nil
    #   render_del_yet
    # else
    #   @project = nil
    #   render_404
    # end
  end

  def organization
    id = params[:organization_id] || params[:id]
    @organization = Organization.find_by_name(id) unless id.nil?
    if @organization and can?(current_user, :read_organization, @organization)
      @in_org_flag = true 
      @organization
    elsif @organization.audit_flag.to_i == 3
      @organization = nil
      render_del_yet
    else
      @organization = nil
      render_404
    end
  end

  def target_user
    id = params[:user_id] || params[:id]
    user = User.find(id) unless id.nil?
    if user.nil?
      user = project.creator
    end
    user
  end

  def repository
    @repository ||= project.repository
  rescue => e
    nil
  end

  def add_abilities
    abilities << Ability
  end

  #检查当前用户是否当前项目的成员
  def is_public_or_project_member
    if @project.creator
      user_project = @project.users_projects.where(user_id: current_user.id).first
      if @project.public || user_project.present?
        return true
      end
      #如果是组织项目，则取得项目所有团队，挑选出来包含此用户的团队权限。
    elsif @project.organization
      @project.organization_teams_projects.each do|team_project|
        if team_project.organization_team.users.include? current_user
         return true
        end
      end
    end
    false
  end

  def authorize_project!(action)
    if project and  @project.audit_flag.to_i == 3 and !@project.blocked
      render_del_yet
    end
    # -Some difference between CODE Enterprise and CODE Community
    # -If user not login. CODE Enterprise will require user to login to view the project page.
    # -CODE Community Just tell client that you are access_denied!
    if current_user.blank? || current_user.id.blank? #user not login or not exists.
      return require_login? unless Gitlab::SystemConfigsUtil.isCommunity
    end
    unless can?(current_user, action, @project)
      #如果用户有项目的部分权限，则在没全下的页面提示没有权限
      if is_public_or_project_member
        flash[:error] = I18n.t("project.permision_prohibit")
        redirect_to project_graph_path(@project, @ref || @repository.root_ref)
      else
        access_denied!
      end
    end
  end

  def authorize_user!(action)
    return access_denied! unless can?(current_user, action, target_user)
  end

  def authorize_code_access!
    return access_denied! unless can?(current_user, :download_code, project)
  end

  def authorize_create_team!
    return access_denied! unless can?(current_user, :create_team, nil)
  end

  # def authorize_manage_user_team!
  #   return access_denied! unless user_team.present? && can?(current_user, :manage_user_team, user_team)
  # end
  #
  def authorize_admin_user_team!
    return access_denied! unless user_team.present? && can?(current_user, :admin_user_team, user_team)
  end

  def authorize_organization!(action)
    return access_denied! unless can?(current_user, action, @organization)
  end

  def authorize_snippet!(action)
    return access_denied! unless can?(current_user, action, @snippet)
  end

  def authorize_task!(action)
    return access_denied! unless can?(current_user, action, @task)
  end

  def authorize_camp_proposal!(action)
    return access_denied! unless can?(current_user, action, @proposal)
  end

  def authorize_camp_org!(action)
    org = @camp_org || CampOrg.new
    return access_denied! unless can?(current_user, action, org)
  end

  def access_denied!
    render_404
  end

  # def not_found!
    render_404
  # end

  def git_not_found!
    render_404
  end

  def method_missing(method_sym, *arguments, &block)
    if method_sym.to_s =~ /^authorize_((.*)organization)!$/
      authorize_organization!($1.to_sym)
    elsif method_sym.to_s =~ /^authorize_(.*_snippet)!$/
      authorize_snippet!($1.to_sym)
    elsif method_sym.to_s =~ /^authorize_(.*_task)!$/
      authorize_task!($1.to_sym)
    elsif method_sym.to_s =~ /^authorize_(.*_camp_proposal)!$/
      authorize_camp_proposal!($1.to_sym)
    elsif method_sym.to_s =~ /^authorize_(.*)_camp_org!$/
      authorize_camp_org!($1.to_sym)
    elsif method_sym.to_s =~ /^authorize_(.*)!$/
      authorize_project!($1.to_sym)
    elsif method_sym.to_s =~ /^auth_user_(.*)!$/
      authorize_user!($1.to_sym)
    else
      super
    end
  end

  def render_drb_server_time_out
    store_pre_url
    render file: Rails.root.join("public", "drb_server_time_out"), layout: false, status: "500"
  end

  def render_404
    store_pre_url
    render file: Rails.root.join("public", "404"), layout: false, status: "404"
  end

  def render_403
    store_pre_url
    render file: Rails.root.join("public", "403"), layout: false, status: "403"
  end

  def render_500
    store_pre_url
    render file: Rails.root.join("public", "500"), layout: false, status: "500"
  end

  def render_del_yet
    render file: Rails.root.join("public", "del_yet"), layout: false, status: "500"
  end


  def require_non_empty_project
    # redirect_to @project if @project.empty_repo? && !@project.doc_flag
    if @project.empty_repo? && !@project.doc_flag
      redirect_to project_tree_path(@project, @project.default_branch.present? ? @project.default_branch : "master")
    end
  end

  def no_cache_headers
    response.headers["Cache-Control"] = "no-cache, no-store, max-age=0, must-revalidate"
    response.headers["Pragma"] = "no-cache"
    response.headers["Expires"] = "Fri, 01 Jan 1990 00:00:00 GMT"
  end

  def dev_tools
    Rack::MiniProfiler.authorize_request
  end

  def set_language
    # -优先从数据库中查询用户设置的语言
    if Gitlab::SystemConfigsUtil.multi_language?
      session[:locale_language] = @current_user.default_language if @current_user.default_language.present? and session[:locale_language].to_s == ""

      if session[:locale_language].to_s == ""
      elsif I18n.locale.to_s != session[:locale_language]
        I18n.locale = session[:locale_language]
      end
    else
      I18n.locale = 'zh-CN'
    end


  end

  def default_headers
    headers['X-Frame-Options'] = 'DENY'
    headers['X-XSS-Protection'] = '1; mode=block'
  end

  def statistic_project_visit
    if @project && @project.id && @project.doc_flag
      session[:visit_project] ||= [] 
      visit_project_ids = session[:visit_project]
     
      unless visit_project_ids.include?(@project.id)
        @project.visit_count = @project.visit_count + 1
        @project.save

        visit_project_ids << @project.id
        session[:visit_project] = visit_project_ids
      end
    end
  end

  def get_set_cache_expires_in
    count = Gitlab.config.cache.expires_in.count
    unit = Gitlab.config.cache.expires_in.unit
    count.to_i.send(unit)
  end

  def current_branch(project)
    case params[:controller]
      when "tree"
        params[:id][/^[^\/]+/]
      else
        params[:ref] || params[:id] || project.default_branch || "master"
    end
  end

  def slice_blob_array(blobs)
    return nil if !blobs
    if blobs.length > 3000
      blobs = blobs[0...3000]
      flash[:warning] = I18n.t("warnings.messages.list_to_big")
    end
    return blobs
  end

  def check_visit_snippets
    @can_visit_webide = false
    @can_visit_webide = WebideTester.where(username: @current_user.username).present?
  end




end