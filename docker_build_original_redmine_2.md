
docker-entrypoint.sh

slack連携を入れる
```
		# s3 plugin
                apt-get install git -y
                cd /usr/src/redmine
                git clone git://github.com/ka8725/redmine_s3.git plugins/redmine_s3
                rm -Rf plugins/redmine_s3/.git

		# slack plugin
		cd /usr/src/redmine
		git clone https://github.com/sciyoshi/redmine-slack.git plugins/redmine_slack
		rm -Rf plugins/redmine_slack/.git

		# theme
    git clone https://github.com/hardpixel/minelab.git public/themes/minelab

		#
    bundle install --without development test

		set -- gosu redmine "$@"
		;;
esac

exec "$@"
```
