# Secure Rails

Everyone writing code must be responsible for security. :lock:

Start with the [Rails Security Guide](https://guides.rubyonrails.org/security.html) to see how Rails protects you.

Also, check out [this guide](https://ankane.org/sensitive-data-rails) for securing sensitive data.

## Best Practices

### Secrets ✅

- Keep secret tokens out of your code - `ENV` variables are a good practice ✅

  **Why:** You don’t want your version control host, CI provider, or any other service with access to your code to have access to your secrets. If one of these services is compromised, or a single developer’s account on one of these services is compromised, you don’t want to lose your secrets.

### SQL Injection 🔍 <i>(We need to audit for this.)</i>

- Even with ActiveRecord, SQL injection is still possible if misused

  ```ruby
  User.group(params[:column])
  ```

  is vulnerable to injection. [Learn about other methods](https://rails-sqli.org)

  **Why:** [This](https://guides.rubyonrails.org/security.html#sql-injection) explains it well

### Host Header Injection 👈

- Prevent [host header injection](http://carlos.bueno.org/2008/06/host-header-injection.html) - add the following to `config/environments/production.rb`

  ```ruby
  config.action_controller.default_url_options = {host: "www.yoursite.com"}
  config.action_controller.asset_host = "www.yoursite.com"
  ```

  **Why:** An attacker can pass a bad host header. If your app uses caching, this bad host may be cached and served to other users (this can happen with `*_url` helpers).

### Data in Transit 👈

- Protect all data in transit with HTTPS - you can get free SSL certificates from [Let’s Encrypt](https://letsencrypt.org/)

  Add the following to `config/environments/production.rb`

  ```ruby
  config.force_ssl = true
  ```

  **Why:** So attackers can’t eavesdrop or modify pages

- Add your domain to the [HSTS Preload List](https://hstspreload.org/)

  ```ruby
  config.ssl_options = {hsts: {subdomains: true, preload: true, expires: 1.year}}
  ```

  **Why:** If someone visits your website over HTTP, even if you have an HTTPS redirect, an attacker can perform a middleperson attack. [sslstrip](https://avicoder.me/2016/02/22/SSLstrip-for-newbies/) is a popular tool for this. The preload list ships with the browser and instructs it to always use HTTPS for specific domains.

### Data at Rest 👈

- Protect sensitive database fields with application-level encryption - use a library like [Lockbox](https://github.com/ankane/lockbox) 👈 <strike>or [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted)</strike> and possibly [KMS Encrypted](https://github.com/ankane/kms_encrypted)

  **Why:** This protects sensitive data if the database or a database backup is compromised

- Protect sensitive files with application-level encryption - use a library like [Lockbox](https://github.com/ankane/lockbox) 🚫 <i>(We upload direct to S3 so we can't do this.)</i>

  **Why:** This protects sensitive data if file storage is compromised, or if someone accidentally makes an S3 bucket public

- Make sure sensitive request parameters aren’t logged 🤔 <i>(Need to think of which parameters this might be by default.)</i>

  ```ruby
  Rails.application.config.filter_parameters += [:credit_card_number]
  ```

  Use [Logstop](https://github.com/ankane/logstop) as an additional line of defense

  **Why:** You don’t want sensitive data in your log files if they are compromised

### Authentication

- Use a trusted library like [Devise](https://github.com/plataformatec/devise) for authentication (see [Hardening Devise](https://ankane.org/hardening-devise) if applicable) ✅

  **Why:** Secure authentication is hard. Use a library that’s battle-tested. Don’t roll your own.

- Notify users of password changes 👈

  **Why:** So users are aware if someone tries to hijack their account

- Notify users of email address changes - send an email to the old address 👈

  **Why:** So users can’t silently hijack the account by changing the email, then the password

- Rate limit login attempts by IP with [Rack Attack](https://github.com/kickstarter/rack-attack) 👈

  **Why:** To slow down [credential stuffing](https://en.wikipedia.org/wiki/Credential_stuffing) attacks

- Log all successful and failed login attempts and password reset attempts (check out [Authtrail](https://github.com/ankane/authtrail) if you use Devise) 👈

  **Why:** So you have an audit trail when accounts are compromised. You can also use this information to detect compromised accounts.

- Rails has a number of gems for [authorization](https://www.ruby-toolbox.com/categories/rails_authorization) - we like [Pundit](https://github.com/elabs/pundit) ✅ (We use CanCanCan for this.)

  **Why:** To prevent users from accessing unauthorized data

### Browser Caching

- Set `autocomplete="off"` for sensitive form fields, like credit card number 🤔 <i>Need to think which fields this might be for us.</i>

  **Why:** So other users of the browser can’t access this saved information

- Ask the browser [not to cache pages](https://stackoverflow.com/a/748646) with sensitive information 👈 <i>(We need this on the API keys page.)</i>

  ```ruby
  response.headers["Cache-Control"] = "no-cache, no-store, max-age=0, must-revalidate"
  response.headers["Pragma"] = "no-cache"
  response.headers["Expires"] = "Sat, 01 Jan 2000 00:00:00 GMT"
  ```

  **Why:** So other users of the browser can’t click the back button and view sensitive information

### Data Leakage

- Ask search engines not to index pages with secret tokens in the URL 🤔 <i>I don't think this would apply to any pages we have.</i>

  ```html
  <meta name="robots" content="noindex, nofollow">
  ```

  **Why:** So search engines don’t index (and therefore expose) the tokens

### Cross-Site Scripting (XSS)

- Use `json_escape` when passing variables to JavaScript, or better yet, a library like [Gon](https://github.com/gazay/gon) 🔍 <i>(We need to audit for this.)</i>

  ```erb
  <script>
    var currentUser = <%= raw json_escape(current_user.to_json) %>;
  </script>
  ```

  **Why:** To prevent cross-site scripting (XSS)

- [Be careful](https://product.reverb.com/2015/08/29/stay-safe-while-using-html_safe-in-rails/) with `html_safe` 🔍 <i>(We need to audit for this with the `html` attribute partial.)</i>

  **Why:** It bypasses escaping

- Don’t use assets from a public CDN, as this creates unnecessary availability and security risk

  **Why:** This adds another attack vector for an attacker

## Open Source Tools

- [Brakeman](https://github.com/presidentbeef/brakeman) is a great static analysis tool - it scans your code for vulnerabilities 👈
- [bundler-audit](https://github.com/rubysec/bundler-audit) checks for vulnerable versions of gems 👈

  ```sh
  gem install bundler-audit
  bundle audit check --update
  ```

  To fix `Insecure Source URI` issues with the `github` option, add to the top of your `Gemfile`:

  ```ruby
  git_source(:github) do |repo_name|
    repo_name = "#{repo_name}/#{repo_name}" unless repo_name.include?("/")
    "https://github.com/#{repo_name}.git"
  end
  ```

  And run `bundle install`.

- [npm audit](https://docs.npmjs.com/getting-started/running-a-security-audit) checks for vulnerable versions of JavaScript packages (if you use `package.json`) 👈
- [git-secrets](https://github.com/awslabs/git-secrets) prevents you from committing sensitive info

  ```ruby
  brew install git-secrets
  git secrets --register-aws --global
  git secrets --install
  git secrets --scan
  ```

## Mailing Lists

Subscribe to [ruby-security-ann](https://groups.google.com/forum/#!forum/ruby-security-ann) to get security announcements for Ruby, Rails, Rubygems, Bundler, and other Ruby ecosystem projects.

## Services

- [Observatory](https://observatory.mozilla.org) scans your site for best practices
- [Hakiri](https://hakiri.io/) monitors for dependency and code vulnerabilities
- [CodeClimate](https://codeclimate.com/) provides a hosted version of static analysis
- [HackerOne](https://hackerone.com/) allows you to enlist hackers to surface vulnerabilities

## Additional Reading

- [Rails’ Insecure Defaults](https://codeclimate.com/blog/rails-insecure-defaults/)
- [The Inadequate Guide to Rails Security](https://blog.honeybadger.io/ruby-security-tutorial-and-rails-security-guide/)
- [The Matasano Crypto Challenges](https://cryptopals.com/)

## Contributing

Have other good practices? Know of more great tools? [Help make this guide better for everyone](https://github.com/ankane/secure_rails/issues/new).

Also check out [Production Rails](https://github.com/ankane/production_rails).
