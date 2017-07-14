desc 'load all required gems and dependencies'
task :dependencies do
  require 'rubygems'
  require 'yaml'
  require 'bundler'
  Bundler.require
end

task establish_connection: :dependencies do
  ActiveRecord::Base.establish_connection adapter: 'sqlite3', database: 'db/orders.sqlite3'
end

desc 'load the environment'
task environment: :dependencies do
  Rake::Task['establish_connection'].execute
  require './lib/amazon/order'
  require './lib/amazon/shipment'
  require './lib/amazon/shipment_order'
  require './lib/amazon/order_importer'
end

desc 'load an interactive console'
task console: :environment do
  require 'pry'
  ARGV.clear
  Pry.start
end

desc 'display some interesting stats about your order history'
task get_latests: :environment do
  include ActiveSupport::NumberHelper
  include ActionView::Helpers::DateHelper
  include Amazon

  begin
      slack = YAML.load_file('./config/slack.yml').with_indifferent_access
      slack['url']
  rescue 
      puts "\nHmm... no Slack URL found.\n".red
      Rake::Task['slack_config:generate'].execute
      retry
  end
  date_time = Time.now
  puts "Starting sending to Slack: #{date_time} "  
  i = 0
  Order.all.each do |order, hsh|
	if order.sent_to_slack
	  next
	end
	begin
		total = "$#{order.line_items['grand_total']}"
		date = order.date.strftime("%B %d, %Y")
		message =  "Amazon.com Order Number: #{order.id} Placed: #{date} Total: #{total}"
		json = {text: message}.to_json
	rescue Exception => e 
	  puts "Order ##{order.order_id.green} skipped because of #{e.message}"
	  order.set_slack
	  order.save!
	  next
	end

	## Posting to slack:
	uri = URI(slack['url'])
	http = Net::HTTP.new(uri.host, uri.port)
	http.use_ssl = true
	http.verify_mode = OpenSSL::SSL::VERIFY_NONE
	req = Net::HTTP::Post.new(uri.to_s)
	req['Content-Type'] = 'application/json'
	req['User-Agent'] = 'Mac Safari'
	req.body = json
	print "sending Order ##{order.order_id.green}   to slack ... "
	res = http.request(req)
	if res.body == "ok"
	  puts "response #{res.body}"
	  order.set_slack
	  order.save!
	  i = i + 1
	else
	  puts "response failed"
	  puts "#{res.body.dump}"
	end
  end
  puts "Sent #{i} orders"

end

task stats: :environment do
  include ActiveSupport::NumberHelper
  include ActionView::Helpers::DateHelper
  include Amazon

  if Order.count == 0
    puts "ERROR: Import some orders first with #{'rake orders:fetch'.yellow}!\n\n"
    next
  end

  def format_row(row_or_first, second = nil)
    arr                   = second ? [row_or_first, second] : row_or_first
    title                 = arr.first.to_s.blue
    value                 = arr.second.yellow
    parenthetical_content = value[/\(.*?\)/]
    value                 = value.gsub(parenthetical_content, parenthetical_content.green) if parenthetical_content
    [title, value]
  end

  orders_by_calendar_month = Order.all.each_with_object({}) do |order, hsh|
    hsh[order.date.beginning_of_month] ||= []
    hsh[order.date.beginning_of_month] << order
  end

  orders_by_calendar_month_sorted = orders_by_calendar_month.keys.each_with_object({}) do |calendar_month, hsh|
    hsh[calendar_month] = orders_by_calendar_month[calendar_month].count
  end.sort_by { |date, orders| orders }.reverse

  amount_by_calendar_month = orders_by_calendar_month.each_with_object({}) do |month_and_orders, hsh|
    hsh[month_and_orders.first] = month_and_orders.last.map(&:grand_total).sum
  end.sort_by { |date, amount| amount }.reverse

  orders_by_month_number = Order.all.each_with_object({}) do |order, hsh|
    hsh[order.date.month] ||= []
    hsh[order.date.month] << order
  end

  order_totals_by_month = orders_by_month_number.keys.each_with_object({}) do |month_number, hsh|
    hsh[Date::MONTHNAMES[month_number]] ||= 0
    hsh[Date::MONTHNAMES[month_number]] += orders_by_month_number[month_number].map(&:grand_total).sum
  end.sort_by { |month, orders| orders }.reverse

  order_counts_by_month = orders_by_month_number.keys.each_with_object({}) do |month_number, hsh|
    hsh[Date::MONTHNAMES[month_number]] = orders_by_month_number[month_number].count
  end.sort_by { |month, amount| amount }.reverse

  first_order = Order.order('date ASC').first

  table = Terminal::Table.new title: 'Your Amazon.com Stats'.red do |t|
    t << format_row('Customer Since', "#{first_order.date.strftime('%B %Y')} (#{distance_of_time_in_words_to_now first_order.date})")
    t << format_row('Total Orders', number_to_human(Order.count))
    t << format_row('Amount Spent', number_to_currency(Order.sum :grand_total))
    t << format_row('Average Amount Spent per Order', number_to_currency(Order.average :grand_total))
    t << :separator
    t << format_row('Cumulative Month with Most Orders', "#{order_counts_by_month.first.first} (#{order_counts_by_month.first.last.to_s.green} #{'orders'.green})")
    t << format_row('Cumulative Month with Most Spent', "#{order_totals_by_month.first.first} (#{number_to_currency(order_totals_by_month.first.last).green})")
    t << :separator
    t << format_row('Calendar Month with Most Orders', "#{orders_by_calendar_month_sorted.first.first.strftime '%B %Y'} (#{orders_by_calendar_month_sorted.first.last.to_s.green} #{'orders'.green})")
    t << format_row('Calendar Month with Most Spent', "#{amount_by_calendar_month.first.first.strftime '%B %Y'} (#{number_to_currency(amount_by_calendar_month.first.last).green})")
  end

  puts "\n"
  puts table
  puts "\n\n"
end

namespace :config do
  desc 'generate a config/account.yml file with your Amazon.com credentials'
  task generate: :dependencies do
    puts "\nNOTE: The credentials you enter\nhere are stored in #{'PLAIN TEXT'.red}.\n\n"
    puts "The credentials will be stored in:\n#{File.expand_path('./config/account.yml').to_s.red}\n\n"

    email = ask 'Amazon.com Account Email: '
    password = ask('Amazon.com Account Password (hidden): ') { |q| q.echo = '*' }

    # confirmation; may or may not want ...
    password_confirm = ask('Confirm Amazon.com Account Password (hidden): ') { |q| q.echo = '*' }
    password = nil unless password == password_confirm

    puts "\n"

    if password.blank? || email.blank?
      puts "No luck with that; please try again.\n\n"
      next
    end

    File.open('./config/account.yml', 'w') { |f| f.puts({ email: email, password: password }.to_yaml) }
    puts "Wrote values to #{File.expand_path('./config/account.yml').to_s}\n\n"
  end
end

namespace :slack_config do
  desc 'generate a config/slack.yml file with your Slack POST url'
  task generate: :dependencies do
    puts "Slack URL will be stored in:\n#{File.expand_path('./config/slack.yml').to_s.red}\n\n"

    slack_url = ask 'Enter Slack URL: '
    puts "\n"

    if slack_url.blank? 
      puts "No luck with that; please try again.\n\n"
      next
    end

    File.open('./config/slack.yml', 'w') { |f| f.puts({ url: slack_url }.to_yaml) }
    puts "Wrote values to #{File.expand_path('./config/slack.yml').to_s}\n\n"
  end
end

namespace :db do
  desc 'wipe all local order data'
  task reset: :establish_connection do
    File.unlink './db/orders.sqlite3' rescue Errno::ENOENT
    Rake::Task['db:migrate'].execute
  end

  desc 'ensure the DB schema is up-to-date'
  task migrate: :establish_connection do
    CURRENT_SCHEMA_VERSION = 1
    if ActiveRecord::Migrator.current_version != CURRENT_SCHEMA_VERSION
      ActiveRecord::Schema.define(version: CURRENT_SCHEMA_VERSION) do
        create_table :orders, id: false, force: true do |t|
          t.string :order_id, null: false, index: true, unique: true
          t.date :date, null: false

          t.float :grand_total, default: 0, null: false
          t.float :items_subtotal, default: 0, null: false
          t.float :estimated_tax_to_be_collected, default: 0, null: false

          t.boolean :gift, default: false, null: false
          t.boolean :completed, default: false, null: false
          t.boolean :sent_to_slack, default: false, null: false
          t.text :line_items
          t.timestamps null: false
        end
        create_table :shipments, force: true, id: false do |t|
          t.string :shipment_id, null: false, index: true, unique: true
          t.string :carrier
          t.string :tracking_number, index: true
          t.string :ship_to
          t.string :shipment_status, null: false, default: 'Unknown'
          t.boolean :delivered, default: false, null: false, index: true
          t.timestamps null: false
        end
        create_table :shipment_orders, force: true, id: false do |t|
          t.string :shipment_id, null: false, index: true
          t.string :order_id, null: false, index: true
        end
      end
    end
  end
end

namespace :orders do

  desc 'log in to amazon and fetch all orders'
  task fetch: :environment do
    date_time = Time.now
    puts "Starting fetching from Amazon: #{date_time} "  
    begin
      puts "Loading account settings ..."
      account = YAML.load_file('./config/account.yml').with_indifferent_access
    rescue Errno::ENOENT
      puts "\nHmm... no settings found.\n".red
      puts "Let's create a configuration file!".green
      Rake::Task['config:generate'].execute
      retry
    end

    agent   = Mechanize.new { |a| a.user_agent_alias = 'Mac Safari' }
    puts "Loading #{'Amazon.com'.yellow}..."
    agent.get('https://www.amazon.com/') do |page|
      puts "Loading the login page.."
      login_page = agent.click(page.link_with( :text => 'Hello. Sign inAccount & ListsSign inAccount & Lists'   ))
      puts "Filling out the form and logging in..."
      begin
        post_login_page = login_page.form_with(action: %r{ap/signin}) do |f|
          account.each_pair { |k,v| f.send "#{k}=", v }
        end.click_button
	#post_login_page.links.each do |link| 
	#	puts "#{link.text} href: #{link.href}"
	#end
	link = "https://www.amazon.com/gp/css/order-history/ref=nav_youraccount_bnav_ya_ad_orders"
	orders_page = agent.get(link)
        #orders_page = agent.click(post_login_page.link_with(text: 'Your Orders'))
      rescue NoMethodError
        puts "\nLogging in to Amazon.com failed.\n".red
        puts "Try running #{'rake config:generate'.yellow} and updating your credentials."
        next
      end

      puts "Logged in successfully -- loading up your orders!\n"

      begin
        Amazon::Order.count
      rescue ActiveRecord::StatementInvalid
        Rake::Task['db:migrate'].execute
      end

      cancelled_page = agent.click(orders_page.link_with(  href: %r{oh_aui_menu_cancelled} ))

      ## New orders
      years = orders_page.search('form#timePeriodForm select[name=orderFilter] option[value^="year-"]').map do |option_tag|
        option_tag.content.strip.to_i
      end
      years.each do |year|
        puts "Looking up orders from #{year.to_s.blue}..."
        orders_page = orders_page.form_with(id: 'timePeriodForm') do |f|
          f.orderFilter = "year-#{year}"
        end.submit
        page = 0
        catch :no_more_orders do
          loop do
            puts "  Parsing orders from #{year.to_s.blue} (page #{(page+1).to_s.red})"
            orders_page.search('#ordersContainer > .order').each do |node|
              order = Amazon::OrderImporter.import(node, agent)
              action = order.skipped? ? 'skipped' : nil
              ( action = order.new_order? ? 'imported' : 'updated' ) if action.nil?
              puts "    #{action.titleize.yellow} Order ##{order.order_id.green} (#{order.reload.shipments.count} shipments)"
            end
            begin
              last_pagination_element = orders_page.search('ul.a-pagination li.a-last')[0]
              link = last_pagination_element.css('a')[0]
            rescue NoMethodError
              throw :no_more_orders
            end
            throw :no_more_orders unless link
            orders_page = agent.click(link)
            page += 1
          end
        end
      end



#	puts "Finished for current orders"
#	## cancelled orders
#	page = 0
#	puts "Looking up for cancelled orders ..."
#	catch :no_more_orders do	
#	  loop do
#            cancelled_page.search('#ordersContainer > .order').each do |node|
#   	      order = Amazon::OrderImporter.import(node, agent)
#              action = order.skipped? ? 'skipped' : nil
#              ( action = order.new_order? ? 'imported' : 'updated' ) if action.nil?
#              puts "    #{action.titleize.yellow} Order ##{order.order_id.green} (#{order.reload.shipments.count} shipments)" 
#	    end
#	    begin
#              last_pagination_element = cancelled_page.search('ul.a-pagination li.a-last')[0]
#              link = last_pagination_element.css('a')[0]
#            rescue NoMethodError
#              throw :no_more_orders
#            end
#	    throw :no_more_orders unless link
#            cancelled_page  = agent.click(link)
#            page += 1
#          end 
#	end
	


      puts "\nYippee! All done."
    end
  end
end
