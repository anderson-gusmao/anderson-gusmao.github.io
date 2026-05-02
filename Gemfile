source "https://rubygems.org"

# Use o gem github-pages para garantir que as versões sejam compatíveis com o GitHub
gem "github-pages", group: :jekyll_plugins

# Tema padrão
gem "minima", "~> 2.5"

# Necessário para Ruby 3.0+
gem "webrick", "~> 1.7"

# Dependências para Windows (ignoradas em outros sistemas)
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]
