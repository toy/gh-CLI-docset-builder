exec 'rake' if __FILE__ == $0

require 'json'
require 'nokogiri'
require 'fspath'

FSPath.class_eval do
  def done = self / '.done'

  def carefull_write(content)
    content = content&.to_s

    return if size? && read == content

    temp_file_path(dirname.mkpath) do |tmp|
      tmp.write(content)
      tmp.rename(self)
    end
  end
end

module GHCLIDocBuilder
  include Rake::DSL
  extend Rake::DSL

  NAME = 'gh-CLI'

  CLEAN_PATHS = [
    DOWNLOAD_PATH = FSPath('download'),
    HTML_PATH = FSPath('html') / NAME,
    DASHING_CONFIG_PATH = FSPath("#{NAME}.json"),
    DOCSET_PATH = FSPath("#{NAME}.docset"),
    ARCHIVE_PATH = FSPath("#{NAME}.docset.tgz"),
    RESULT_PATH = FSPath('results') / NAME,
  ]
  VERSION_PATH = DOWNLOAD_PATH / 'version'
  DOCSET_META_PATH = RESULT_PATH / 'docset.json'

  class << self
    def define_tasks
      dir DOWNLOAD_PATH => [] do
        latest_release_url = `curl -Ls -o /dev/null -w '%{url_effective}' https://github.com/cli/cli/releases/latest`
        version = latest_release_url[%r{/v(\d+(?:\.\d+)+)\z}, 1]
        fail "no version from #{latest_release_url}" unless version

        sh(*%W[
          wget
          --no-parent
          --recursive
          --page-requisites
          --convert-links
          --adjust-extension
          --no-host-directories
          --directory-prefix=#{DOWNLOAD_PATH}
          --quiet
          --show-progress
          https://cli.github.com/manual/
        ])

        VERSION_PATH.carefull_write(version.sub(/\.0\z/, ''))
      end

      dir HTML_PATH => DOWNLOAD_PATH.done do
        DOWNLOAD_PATH.find do |src_path|
          next if src_path == DOWNLOAD_PATH.done

          dst_path = HTML_PATH + src_path.relative_path_from(DOWNLOAD_PATH)

          if src_path.directory?
            mkdir_p dst_path
            next
          end

          if src_path.extname == '.html'
            doc = Nokogiri::HTML(src_path.read)

            doc.at('html').add_class('dash-ignore-dark-mode')

            head = doc.at('head')
            head.children.each do |child|
              child.remove if child.attributes.any?{ |_, attribute| attribute.value =~ %r{\Ahttps?://} }
            end
            style = Nokogiri::XML::Node.new('style', doc)
            style.content = <<~CSS
              a.external::after {
                content: "↗";
                padding-left: 0.25em;
              }
            CSS
            head.add_child(style)

            main = doc.at('main').unlink
            main.remove_class('col-lg-10')
            main.first_element_child.remove_class('container-lg')

            body = doc.at('body')
            body.children = main

            first_header = doc.at('h1, h2')
            if first_header.name == 'h2' && first_header.text =~ /\Agh/
              first_header['data-element-type'] = 'Command'
            else
              doc.search('h1').each{ |h| h['data-element-type'] = 'Guide' }
              doc.search('h2').each{ |h| h['data-element-type'] = 'Section' }
            end

            doc.search('a[href]').each do |a|
              a.add_class('external') if a['href'] =~ %r{\Ahttps?://}
            end

            dst_path.carefull_write(doc)
          else
            cp src_path, dst_path
          end
        end
      end

      file DASHING_CONFIG_PATH => HTML_PATH.done do
        DASHING_CONFIG_PATH.carefull_write(dashing_config)
      end

      dir DOCSET_PATH => [DASHING_CONFIG_PATH, HTML_PATH.done] do
        mkdir_p DOCSET_PATH.dirname

        sh(*%W[dashing build --source #{HTML_PATH} --config #{DASHING_CONFIG_PATH}])

        %w[icon.png icon@2x.png].each do |icon_basename|
          ln icon_basename, DOCSET_PATH
        end
      end

      file ARCHIVE_PATH => DOCSET_PATH.done do
        mkdir_p ARCHIVE_PATH.dirname

        sh(*docker_run_args('debian') + %W[
          tar
          --directory=#{DOCSET_PATH.dirname}
          --exclude=.DS_Store
          -cvzf
          #{ARCHIVE_PATH}
          #{DOCSET_PATH.basename}
        ])
      end

      results = {
        RESULT_PATH / ARCHIVE_PATH.basename => ARCHIVE_PATH,
        **%w[README.markdown icon.png icon@2x.png].to_h do |basename|
          [RESULT_PATH / basename, basename]
        end,
      }

      file DOCSET_META_PATH => results.values do
        rm_rf RESULT_PATH
        mkdir_p RESULT_PATH

        results.each do |dst, src|
          ln src, dst
        end

        DOCSET_META_PATH.carefull_write(docset_meta)
      end

      task default: DOCSET_META_PATH

      task :clean do
        rm_rf CLEAN_PATHS
      end
    end

  private

    def dir(arg)
      fail ArgumentError, 'expected Hash with 1 pair' unless arg.is_a?(Hash) && arg.length == 1

      dir, dependencies = arg.first
      done = dir.done

      file done => dependencies do
        rm_rf dir

        yield

        touch done
      end
    end

    def dashing_config
      JSON.pretty_generate({
        name: NAME,
        package: NAME,
        index: "#{HTML_PATH}/manual/index.html",
        externalURL: 'https://cli.github.com/manual/',
        selectors: %w[Guide Section Command].to_h do |entry_type|
          [%Q{[data-element-type="#{entry_type}"]}, entry_type]
        end,
      })
    end

    def docset_meta
      JSON.pretty_generate({
        name: NAME,
        version: VERSION_PATH.read,
        archive: ARCHIVE_PATH.basename,
        author: {
          name: 'Ivan Kuchin',
          link: 'https://github.com/toy',
        },
      })
    end

    def docker_run_args(image)
      %W[
        docker run
        --user #{Process.uid}:#{Process.gid}
        --rm
        -i
        -v #{Dir.pwd}:/here
        -w /here
        #{image}
      ]
    end
  end
end

GHCLIDocBuilder.define_tasks
