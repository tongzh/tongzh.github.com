## About
This is source code of [my tech blog](http://tongzh.github.com)

## Usage
### How to run the server in localhost
Firstly run `gem install jekyll` to install jekyll gem, then run `jekyll server` to lanuch the server, you can browse it at [http://localhost:4000]() now.

You can also use `jekyll server -D` to launch the server in draft mode and preview the drafts.

Or use `jekyll server -Dw` to launch in draft mode and watch your file changes.

### Create posts or pages
Run `rake post title='Hello World'` to create a new post.

Run `rake page name='about.md'` to create a new page, or `rake page name='pages/about.md'` to create a nested page.

After editing your posts or pages, then use git commands to push it to remote repositoy.

### Create drafts
Just create a .md file in `_drafts` folder, then you can edit posts in draft mode.

For markdown syntax, please check [https://github.com/younghz/Markdown].

## Contact me

[Mail to me](mailto:zhangtong00@gmail.com)
