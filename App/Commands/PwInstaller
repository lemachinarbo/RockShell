<?php

namespace RockShell;

use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\DomCrawler\Crawler;
use Symfony\Component\DomCrawler\Field\ChoiceFormField;
use Symfony\Component\HttpClient\HttpClient;

class PwInstaller extends Command
{
  /** @var HttpBrowser */
  protected $browser;

  private $host;
  private $skipWelcome = false;
  private $skipNextConfirm = false;
  private $stepCount = 0;

  // Set this to false for interactive by default; true only with --lazy
  private $lazy = false;

  // All installer defaults in one place
  private $lazyDefaults = [
    // Host/General
    // 'host' => 'foo.ddev.site', // host is autodetected but can be overridden
    'debug' => false,
    'download_processwire' => true,
    'processwire_version' => 'dev',
    'download_rockfrontend' => false,
    'profile' => 'site-blank', // site-rockfrontend requires download_rockfrontend to be true
    // Database
    'dbName' => 'db',
    'dbUser' => 'db',
    'dbPass' => 'db',
    'dbHost' => 'db',
    'dbCon'  => 'Hostname',
    'dbPort' => 3306,
    'dbCharset' => 'utf8mb4',
    'dbEngine' => 'InnoDB',
    'dbTablesAction' => 'remove', // remove existing tables if pw is already installed
    // Admin
    'admin_name' => 'adm',
    'username' => 'ddevadmin',
    'userpass' => 'ddevadmin',
    'userpass_confirm' => 'ddevadmin',
    'useremail' => 'admin@example.com',
    // Site
    'timezone' => 'America/Bogota', // find yours at https://www.php.net/manual/en/timezones.php
    'debugMode' => 1,
  ];

  public function config()
  {
    $this
      ->setDescription("Install ProcessWire (lazy mode)")
      ->addOption("host", null, InputOption::VALUE_REQUIRED, "Hostname of your new site")
      ->addOption('profile', null, InputOption::VALUE_REQUIRED, "Site-Profile to install")
      ->addOption('debug', 'd', InputOption::VALUE_NONE, "Enable debug mode")
      ->addOption('step', null, InputOption::VALUE_REQUIRED, "Step")
      ->addOption('timezone', 't', InputOption::VALUE_REQUIRED, "Timezone (int or string)")
      ->addOption('remove', 'r', InputOption::VALUE_NONE)
      ->addOption('ignore', 'i', InputOption::VALUE_NONE)
      ->addOption('url', 'u', InputOption::VALUE_REQUIRED, "Url of the backend")
      ->addOption('name', null, InputOption::VALUE_REQUIRED, "Name of superuser")
      ->addOption('pass', 'p', InputOption::VALUE_REQUIRED, "Password of superuser")
      ->addOption('mail', 'm', InputOption::VALUE_REQUIRED, "Mail-Address of superuser")
      ->addOption("dev", null, InputOption::VALUE_NONE, "Download dev version of pw?")
      ->addOption('lazy', null, InputOption::VALUE_NONE, 'Enable lazy (non-interactive) mode');
  }

  public function handle()
  {
    $this->lazy = $this->option('lazy') ? true : false;

    if ($this->ddevExists() and !$this->ddev()) {
      $this->error("Use ddev ssh to execute this command from within DDEV");
      return self::FAILURE;
    }

    if ($this->wire()) {
      $this->alert("ProcessWire is already installed!");
      return self::SUCCESS;
    }

    $this->browser = new HttpBrowser(HttpClient::create());
    $this->nextStep(true);
    return self::SUCCESS;
  }

  /** ##### steps ##### */

  public function nextStep($reload = false, $noConfirm = false)
  {
    $this->stepCount = $this->stepCount + 1;
    if ($this->stepCount > 50) return; // hard limit to prevent endless loops

    // if a reload is required we fire another request
    if ($reload) $this->browser->request('GET', $this->host('install.php'));

    // get current step from headline
    $step = $this->getStep();
    if (!$step) return false;
    if (is_array($step)) return $this->warn($this->str($step));

    // show errors
    $this->browser->getCrawler()->filter("div.uk-alert")
      ->each(function (Crawler $el) {
        $this->warn($el->text());
      });

    // execute the step
    $method = "step" . ucfirst($step);
    $next = $this->$method();
    if ($this->getLazyValue('debug', false)) {
      $this->warn("Step $step (debug mode) - entering interactive shell");
      $this->pause([
        'browser' => $this->browser,
      ]);
    }

    // next step?
    // if the last step returned false we break
    if ($next !== false) {
      if ($this->skipNextConfirm || $noConfirm) return $this->nextStep();
      if ($this->lazy) return $this->nextStep(); // always continue in lazy mode
      if ($this->confirm("Continue to next step?", true)) return $this->nextStep();
    }
  }

  public function stepWelcome()
  {
    if (!$this->skipWelcome) $this->write('Welcome');
    $this->browser->submitForm('Get Started');
  }

  public function stepProfile()
  {
    $zip = 'https://github.com/baumrock/site-rockfrontend/releases/latest/download/site-rockfrontend.zip';
    $exists = is_dir("site-rockfrontend");
    if (file_exists('site-rockfrontend.zip')) $this->exec('rm site-rockfrontend.zip');
    $download = $this->getLazyValue('download_rockfrontend', true);
    if (!$exists && ($this->lazy ? $download : $this->confirm("Download RockFrontend Site Profile?", true))) {
      $this->write('Downloading ...');
      $this->exec("wget --quiet $zip");
      $this->write('Extracting files ...');
      $this->exec('unzip -q site-rockfrontend.zip');
      $this->nextStep(true, true);
      return;
    }

    $this->newLine();
    $this->write("Install site profile ...");

    $profiles = $this->browser
      ->getCrawler()
      ->filter('select[name=profile] > option')
      ->extract(['value']);
    if (!count($profiles)) {
      $this->error("No profiles found - aborting ...");
      die();
    }
    $profiles = array_values(array_filter($profiles));
    if ($this->lazy) {
      $profile = $this->getLazyValue('profile', $profiles[0]);
    } else {
      // Determine default profile: CLI > lazyDefaults (case-insensitive) > first profile
      $cliProfile = null;
      try {
        $cliProfile = parent::option('profile');
      } catch (\Throwable $th) {}
      $defaultProfile = $profiles[0];
      if ($cliProfile !== null && $cliProfile !== false) {
        $defaultProfile = $cliProfile;
      } else {
        foreach ($profiles as $p) {
          if (trim(strtolower($p)) === trim(strtolower($this->lazyDefaults['profile']))) {
            $defaultProfile = $p;
            break;
          }
        }
      }
      // Robust case-insensitive, trimmed match for default profile
      $defaultIndex = 0;
      foreach ($profiles as $i => $p) {
        if (trim(strtolower($p)) === trim(strtolower($defaultProfile))) {
          $defaultIndex = $i;
          $defaultProfile = $p; // use the actual value from $profiles
          break;
        }
      }
      $profile = $this->choice(
        "Select profile to install [{$defaultProfile}]",
        $profiles,
        $defaultIndex
      );
    }
    $this->write("Using profile $profile ...");
    $this->browser->submitForm("Continue", [
      'profile' => $profile,
    ]);
  }

  public function stepCompatibility()
  {
    $this->write("Checking compatibility ...");
    $errors = 0;
    $this->browser
      ->getCrawler()
      ->filter('div.uk-section-muted > div.uk-container > div')
      ->each(function (Crawler $el) use (&$errors) {
        $text = $el->text();
        $outer = $el->outerHtml();
        if (strpos($outer, 'fa-check')) $this->success($text);
        else {
          $errors++;
          $this->warn($text);
        }
      });
    if ($errors) {
      // In lazy mode, always continue
      if ($this->lazy) {
        $this->skipWelcome = true;
        $this->skipNextConfirm = true;
        return $this->nextStep(true);
      }
      if ($this->confirm("Check again?", true)) {
        $this->skipWelcome = true;
        $this->skipNextConfirm = true;
        return $this->nextStep(true);
      } else {
        if ($this->confirm("Continue Installation?", false)) {
          $this->skipNextConfirm = true;
          $this->browser->submitForm('Continue to Next Step');
        } else {
          $this->warn('Aborting ...');
          die();
        }
      }
    } else {
      $this->skipNextConfirm = false;
      $this->browser->submitForm('Continue to Next Step');
    }
  }

  public function stepDatabase()
  {
    $this->newLine();
    $this->write("Setting the following sections:");
    $this->browser->getCrawler()->filter('h2')->each(function (Crawler $el) {
      $this->write("  " . $el->text());
    });
    $form = $this->fillForm($this->lazyDefaults);
    $this->browser->submitForm('Continue', $form->getValues());
  }

  public function stepAdmin()
  {
    $this->write("Setup admin panel and user");
    $form = $this->fillForm($this->lazyDefaults);
    if ($this->getLazyValue('debug', false) && $this->output->isVeryVerbose()) var_dump($form->getValues());
    $this->browser->submitForm("Continue", $form->getValues());
  }

  public function stepFinish()
  {
    $this->success("Finishing installation ...");
    $this->writeNotes();
    return $this->stepReloadAdmin();
  }

  public function stepReloadAdmin($notice = true)
  {
    if ($notice) $this->info("\nLoading ProcessWire ...");
    chdir($this->app->docroot());
    include "index.php";
    /** @var ProcessWire $wire */
    $url = $this->host($wire->pages->get(2)->url);
    if ($notice) $this->write($url);
    $this->browser->request('GET', $url);

    $notices = 0;
    $this->browser->getCrawler()
      ->filter("li.NoticeMessage")
      ->each(function (Crawler $el) use (&$notices) {
        $notices++;
        $this->write($el->text());
      });
    if ($notices) {
      $this->warn("Reloading ...");
      return $this->stepReloadAdmin(false);
    } else {
      $this->success("\n"
        . "##### INSTALL SUCCESSFUL ######\n"
        . "### powered by baumrock.com ###\n");
      $this->warn("Login: $url");
      die();
    }
  }

  /**
   * Generic value getter for lazy mode
   */
  private function getLazyValue($key, $default = null)
  {
    // CLI option always wins
    $cli = null;
    try {
      $cli = parent::option($key);
    } catch (\Throwable $th) {}
    if ($cli !== null && $cli !== false) return $cli;
    // Lazy mode: use defaults if set
    if ($this->lazy && array_key_exists($key, $this->lazyDefaults)) {
      return $this->lazyDefaults[$key];
    }
    // Fallback to provided default
    return $default;
  }

  /**
   * Normalize a URL: remove :443 for https and :80 for http
   */
  private function normalizeUrl($url)
  {
    $parts = parse_url($url);
    if (!$parts || !isset($parts['scheme'], $parts['host'])) return $url;

    $scheme = $parts['scheme'];
    $host = $parts['host'];
    $port = $parts['port'] ?? null;
    $path = $parts['path'] ?? '';

    $isDefaultPort = ($scheme === 'https' && ($port == 443 || $port === null)) ||
      ($scheme === 'http' && ($port == 80 || $port === null));

    if (!$isDefaultPort && $port) {
      return "$scheme://$host:$port$path";
    }

    return "$scheme://$host$path";
  }

  /**
   * Cleans and normalizes the httpHosts array
   */
  private function cleanHttpHostsArray($hosts): array
  {
    if (is_string($hosts)) {
      $hosts = preg_split('/[\s,]+/', $hosts);
    }
    $hosts = array_map('trim', $hosts);
    $hosts = array_filter($hosts);

    $seen = [];
    $cleaned = [];

    foreach ($hosts as $host) {
      // extract bare host if port is 80/443
      if (preg_match('/^(.+):(443|80)$/', $host, $m)) {
        $bare = $m[1];
        // if we've already seen the bare host, skip
        if (isset($seen[$bare])) continue;
        // mark this host as seen but don't add yet
        if (!isset($seen[$host])) $seen[$host] = 'skip';
        continue;
      }

      // if it's a bare host and we've seen the port version, override
      $seen[$host] = 'add';
      $cleaned[] = $host;
    }

    // Add port-specific hosts if their bare host hasn't been added
    foreach ($seen as $host => $action) {
      if ($action === 'skip') $cleaned[] = $host;
    }

    return array_values(array_unique($cleaned));
  }

  /**
   * @return Form
   */
  public function fillForm($defaults = [])
  {
    $form = $this->browser->getCrawler()->filter('.InputfieldForm');
    if (!$form->count()) return $this->error('No form found');
    $form = $form->form();
    $values = $form->getPhpValues();
    $pass = '';
    
    $skipAll = false;
    foreach ($values as $name => $val) {
      if ($skipAll) continue;
      $options = [];
      $field = $form[$name];

      // Defer value resolution: prefer CLI option, then defaults, then form value
      $default = $this->getLazyValue($name, array_key_exists($name, $defaults) ? $defaults[$name] : $val);
      $promptDefault = $default;
      $label = $name;
      
      // Special handling for certain fields
      if ($name == 'timezone') {
        $options = [];
        $phpTimezones = [];
        $this->browser->getCrawler()
          ->filter("select[name=timezone] > option")
          ->each(function (Crawler $el) use (&$options, &$phpTimezones) {
            $phpName = $el->text();
            $label = $phpName;
            $parts = explode("/", $phpName, 2);
            if (count($parts) == 2) {
              $label = $parts[1] . " (" . $parts[0] . ")";
            }
            $options[$el->attr('value')] = strtolower($label);
            $phpTimezones[$el->attr('value')] = $phpName;
          });
        // Try to find the numeric key for the PHP timezone name
        $timezoneKey = null;
        if ($default) {
          foreach ($phpTimezones as $key => $phpName) {
            if (strtolower($phpName) === strtolower($default)) {
              $timezoneKey = $key;
              break;
            }
          }
        }
        $label = "timezone (type 'vienna' for Europe/Vienna to get autocomplete suggestions)";
        if ($this->lazy) {
          if ($timezoneKey !== null) {
            $value = $timezoneKey;
          } else {
            // No valid default, prompt user even in lazy mode
            $chosen = $this->askWithCompletion($label, $options, $default);
            $value = array_search($chosen, $options);
            if ($value === false) $value = '0'; // fallback to UTC
          }
        } else {
          $chosen = $this->askWithCompletion($label, $options, $default);
          $value = array_search($chosen, $options);
          if ($value === false) $value = '0'; // fallback to UTC
          if ($this->output->isVerbose()) $this->write("$name=$value, $chosen");
        }
      } elseif ($name == 'httpHosts') {
        $label = "httpHosts (enter comma separated list)";
        if (!empty($default)) {
          $defaultLines = $this->cleanHttpHostsArray($default);
          $promptDefault = implode("\n", $defaultLines);
        }
        if ($this->lazy) {
          $value = $promptDefault;
        } else {
          $value = $this->askWithCompletion($label, $options, $promptDefault);
        }
        $hosts = is_string($value) ? preg_split('/[\n,]+/', $value) : (array)$value;
        $hosts = array_filter(array_map('trim', $hosts));
        $value = implode("\n", $hosts);
      } elseif ($name == 'admin_name') {
        $label = 'Enter url of your admin interface';
        $promptDefault = $this->getLazyValue('url') ?: $promptDefault;
        if ($this->lazy) {
          $value = $promptDefault;
        } else {
          $value = $this->ask($label, $promptDefault);
        }
        if ($this->output->isVerbose()) $this->write("$name=$value");
      } elseif ($name == 'userpass') {
        // Always use the default from lazyDefaults unless overridden
        $promptDefault = $this->getLazyValue('userpass', $promptDefault);
        do {
          if ($this->lazy) {
            $value = $promptDefault;
          } else {
            $value = $this->ask($name, $promptDefault);
          }
          if (strlen($value) < 6 && !$this->lazy) {
            $this->warn('Password must be at least 6 characters long.');
          }
        } while (strlen($value) < 6 && !$this->lazy);
        $pass = $value; // Set pass immediately for userpass_confirm
        if ($this->output->isVerbose()) $this->write("$name=$value");
      } elseif ($name == 'userpass_confirm') {
        // Always use the just-entered password as default
        if ($this->lazy) {
          $value = $pass;
        } else {
          do {
            $value = $this->ask($name, $pass);
            if ($value !== $pass) {
              $this->warn('Passwords do not match. Please try again.');
            }
          } while ($value !== $pass);
        }
        if ($this->output->isVerbose()) $this->write("$name=$value");
      } elseif ($name == 'remove_items') {
        // Handle removable items checkboxes
        $checkboxes = $this->browser->getCrawler()->filter("input[type=checkbox][name^=remove_items]");
        $selected = [];
        foreach ($checkboxes as $checkbox) {
          $cb = new Crawler($checkbox);
          $value = $cb->attr('value');
          // Try to get the label text (assume label is parent or nearby)
          $label = $cb->ancestors()->filter('label')->count()
            ? trim($cb->ancestors()->filter('label')->text())
            : ($cb->attr('value') ?: '');
          if ($this->lazy) {
            // In lazy mode, select all
            $selected[] = $value;
          } else {
            $ask = $label ?: "Remove $value?";
            if ($this->confirm($ask, true)) {
              $selected[] = $value;
            }
          }
        }
        $form->setValues(['remove_items' => $selected]);
        continue;
      } elseif ($name == 'dbTablesAction') {
        if ($this->option('remove')) $value = 'remove';
        elseif ($this->option('ignore')) $value = 'ignore';
        elseif ($this->lazy) $value = $promptDefault;
        else $value = $this->choice("DB not empty", [
          'remove',
          'ignore',
        ], 0);
        $this->warn("\ndbTablesAction: $value"); // show always
        $skipAll = true;
      } else {
        if ($field instanceof ChoiceFormField) {
          $this->browser
            ->getCrawler()
            ->filter("input[name=$name],select[name=$name] > option")
            ->each(function (Crawler $el) use (&$options) {
              $options[] = $el->attr('value');
            });
        }
        if ($this->lazy) {
          $value = $promptDefault;
        } else {
          $value = $this->askWithCompletion($name, $options, $promptDefault);
        }
        if ($this->output->isVerbose()) $this->write("$name=$value");
      }
      // save password for later confirmation
      if ($name == 'userpass' ) $pass = $value;
      $form->setValues([$name => $value]);
    }
    return $form;
  }


  public function getStep()
  {
    $h1 = $this->browser->getCrawler()->filter('h1');
    $h1 = $h1->count() ? $h1->outerHtml() : '';
    if ($h1 !== '<h1 class="uk-margin-remove-top">ProcessWire 3.x Installer</h1>') {
      $this->write('No ProcessWire Installer found');
      if (is_file($this->app->docroot() . "index.php")) {
        $this->write("");
        $this->error("Found index.php - aborting ...");
        $this->write("");
        die();
      }
      $download = $this->getLazyValue('download_processwire', true);
      if ($this->lazy ? $download : $this->confirm("Download ProcessWire now?", true)) {
        $versions = ['master', 'dev'];
        $version = $this->getLazyValue('processwire_version', 'dev');
        if (!$this->lazy) {
          $version = $this->choice("Which version?", $versions, $version);
        }
        $this->call("pw:download", ['version' => $version]);
        sleep(1);
        return $this->nextStep(true);
      }
      $this->warn("Aborting ...");
      die();
    }

    $headlines = [];
    foreach ($this->browser->getCrawler()->filter('h2') as $h2) {
      $headlines[] = $h2->textContent;
    }
    $headlines = array_reverse($headlines);
    foreach ($headlines as $headline) {
      $headline = trim($headline);
      if ($headline == 'Compatibility Check') return 'compatibility';
      if ($headline == 'Site Installation Profile') return 'profile';
      if (strpos($headline, "Welcome.") === 0) return 'welcome';
      if ($headline == 'Debug mode?') return 'database';
      if ($headline == 'Admin Panel') return 'admin';
      if ($headline == 'Admin Account Saved') return 'finish';
    }
    return $headlines;
  }

  public function host($site)
  {
    $defaulthost = getenv('DDEV_PROJECT') ? getenv('DDEV_PROJECT') . ".ddev.site" : "example.com";
    $site = ltrim($site, "/");
    if ($this->host) {
      $host = $this->host;
    } elseif ($this->option('host')) {
      $host = $this->option('host');
    } elseif ($this->lazy) {
      $host = $this->getLazyValue('host', $defaulthost);
    } else {
      $host = $this->ask('Enter host', $defaulthost);
    }
    $this->host = $host;

    // Ports from environment
    $httpPort = getenv('DDEV_ROUTER_HTTP_PORT');
    $httpsPort = getenv('DDEV_ROUTER_HTTPS_PORT');

    $urlsToCheck = [];
    if (parse_url($host, PHP_URL_PORT) === null) {  // No port in host
      if ($httpsPort) $urlsToCheck[] = $this->normalizeUrl("https://$host:$httpsPort");
      if ($httpPort) $urlsToCheck[] = $this->normalizeUrl("http://$host:$httpPort");
    } else {
      $urlsToCheck = [
        $this->normalizeUrl("https://$host"),
        $this->normalizeUrl("http://$host")
      ];
    }

    foreach ($urlsToCheck as $url) {
      $this->browser->request("GET", $url);
      $status = $this->browser->getInternalResponse()->getStatusCode();
      if ($status === 200) {
        $this->success("Status check for host $url was OK");
        return "$url/$site";
      }
      if ($status === 403) {
        $this->success("Status check for host $url was OK");
        $this->warn("Access is forbidden (403). This may be expected during installation.");
        return "$url/$site";
      }
    }

    $this->error("Your host $host must be reachable via HTTP or HTTPS!");
    $this->error("When using DDEV make sure it is running.");
    exit(1);
  }

  public function writeNotes()
  {
    $this->browser->getCrawler()
      ->filter("div.uk-section-muted > div.uk-container > div")
      ->each(function (Crawler $el) {
        $this->write("  " . $el->text());
      });
  }

  public function findTimezone($t, $options)
  {
    if (is_string($t) and !(int)$t) {
      $t = strtolower($t);
      foreach ($options as $k => $v) {
        if ($v === $t) return $k;
        if (strpos($v, $t) !== false) return $k;
      }
    }
    return $t;
  }
}
