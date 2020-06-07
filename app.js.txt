const express=require('express');
const mysql=require('mysql');

const path=require('path');
const session = require('express-session');



//create connection


const db = mysql.createConnection({
    host:'localhost',
    user:'root',
    password:'viratkohli18',
    database:'elibrary',
    multipleStatements:true,
    
    
});




db.connect((err)=>{
    if(err){
        throw err;
    }
    console.log('connected');
});

//run every time a request is made
const logger=(req,res,next)=>{
    console.log(`${req.protocol}://${req.get('host')}${req.originalUrl}`);
    next();
};

const app=express();
const bodyparser=require('body-parser');
app.use(bodyparser.json());
app.use(express.static(path.join(__dirname,'global')));
app.use(logger);
app.set('view engine','ejs');
app.set('views',path.join(__dirname,'views'));
app.use(session({
	secret: 'secret',
	resave: true,
	saveUninitialized: true
}));
app.use(bodyparser.urlencoded({extended : true}));

//routes
app.get('/home',(req,res)=>{
    
    
        if(req.session.loggedin){
            let sql;
            if(req.session.key){
                sql="Select book.Book_name,author.name,book.price from author join written_by on author.id=written_by.author_id join book on written_by.book_id=book.id where book.Book_name like '%"+req.session.keyword+"%'";
                req.session.key=false;
            }else {
                sql="Select book.Book_name,author.name,book.price,book.version,book.size from author join written_by on author.id=written_by.author_id join book on written_by.book_id=book.id order by book.id";
            }
                db.query(sql,(err,rows,fields)=>{
                if(err) throw err;
                let name,username;
               
                console.log(name,username);
                res.render('home',{
                    rows:rows,
                    username:req.session.username
                    
                
                });
            });
        }else {
            res.send("Please login to view this page");
        }
    
    
    
});
app.post('/pay',(req,res)=>{

    let sql="INSERT INTO `elibrary`.`payment` (`Id`, `Date`, `Total`, `Cart_id`) VALUES (?, (Select Date from cart where id=?), ?, ?);";
    sql+="DELETE FROM cart where id=?;"
    let c=req.session.cart;
    console.log(c);
    req.session.cart++;
    let t=req.session.total;
    req.session.total=0;
    
    db.query(sql,[c,c,t,c],(err,rows,fields)=>{
        
        
        res.redirect('/user');
    });
});
app.get('/cart',(req,res)=>{
    let sql;
            
            
            sql="Select book.id,book.Book_name,book.price from book join added on book.id=added.Book_id where added.id=?";
        
            db.query(sql,[req.session.cart],(err,rows,fields)=>{
                if(err) throw err;
                sql="Select sum(book.price) as total from book join added on book.id=added.Book_id where added.id=?;";
                db.query(sql,[req.session.cart],(err,results,fields)=>{
                    req.session.total=results[0].total;
                    res.render('cart',{
                        rows:rows,
                        total:results[0].total
                        
                    
                    });
                })
            
        });
    
});

app.post('/buy/:id?', (req, res)=> {
    var bookid=req.params.id;
    let sql="INSERT INTO `elibrary`.`added` (`Id`, `Username`, `Book_id`, `Price`) VALUES (?, ?, ?, (Select book.price from book where book.id=?));";
    
    db.query(sql,[req.session.cart,req.session.name,bookid,bookid],(err,rows,fields)=>{
        res.redirect('/user');
    });
    
    
});
app.post('/remove/:id?', (req, res)=> {
    var bookid=req.params.id;
    let sql="DELETE FROM `elibrary`.`added` WHERE added.Book_id=?;";
    
    db.query(sql,[bookid],(err,rows,fields)=>{
        res.redirect('/cart');
    });
    
    
});
app.get('/user',(req,res)=>{
    let v=db.query("Select * from book");
    console.log(req.session.cart);
    
        if(req.session.loggedin){
            let sql;
            if(req.session.key){
                sql="Select book.id,book.Book_name,author.name,book.price from author join written_by on author.id=written_by.author_id join book on written_by.book_id=book.id where book.Book_name like '%"+req.session.keyword+"%'";
                req.session.key=false;
            }else {
                sql="Select book.id,book.Book_name,author.name,book.price,book.version,book.size from author join written_by on author.id=written_by.author_id join book on written_by.book_id=book.id order by book.id";
            }
                db.query(sql,(err,rows,fields)=>{
                if(err) throw err;
                let name,username;
               
                console.log(name,username);
                res.render('user',{
                    rows:rows,
                    username:req.session.username
                    
                
                });
            });
        }else {
            res.send("Please login to view this page");
        }
    
    
    
});

app.get('/log', function(req, res) {
    
	res.render('login');
});

app.post('/out', function(req, res) {
    req.session.loggedin=false;
    req.session.username=undefined;
    req.session.name=undefined;
    let sql="DELETE FROM cart";
    db.query(sql);

    res.redirect('/');
});
app.post('/auth', (req, res)=> {
	var username0 = req.body.admin;
    var password0 = req.body.code;
    
    
	if (username0 && password0) {
		db.query('SELECT * FROM user join admin on user.username=admin.username WHERE user.username = ? AND password = ?', [username0, password0],(error, results, fields)=> {
			if (results.length > 0) {
                console.log(results);
				req.session.loggedin = true;
                req.session.name=results[0].Username;
                req.session.username=results[0].Name;
                req.session.key=false;
                
                res.redirect('/home');
                
				
			} else {
                res.send('Incorrect Username and/or Password!');
                res.end();
			}			
			res.end();
        });
    }else {
		res.send('Please enter Username and Password!');
		res.end();
	}
});
app.post('/member', (req, res)=> {
	
    var username1 = req.body.user;
    var password1 = req.body.pass;
    
	if (username1 && password1) {
		db.query('SELECT * FROM user join elibrary.members on user.username=members.username WHERE user.username = ? AND password = ?', [username1, password1],(error, results, fields)=> {
			if (results.length > 0) {
                
				req.session.loggedin = true;
                req.session.name=results[0].Username;
                req.session.username=results[0].Name;
                req.session.key=false;
                sql="SELECT * from payment";
                db.query(sql,(err,rows,fields)=>{
                let i=rows.length+1;
                req.session.cart=i;
                sql="DELETE FROM added where added.id=?";   
                db.query(sql,[i]);
                
               sql="insert into cart select * from ( select ?, date(now())) as temp where not exists (Select * from cart where cart.id=?) limit 1; " 
                db.query(sql,[i,i]);
                
                
                
                res.redirect('/user');
                    res.end();
                
                });
                
			} else {
                res.send('Incorrect Username and/or Password!');
                res.end();
			}			
			
        });
    } else {
		res.send('Please enter Username and Password!');
		res.end();
	}
});

app.post('/up',(req,res)=>{
    let username=req.body.username;
        
    let fullname=req.body.fullname;
    let contact=req.body.contact;
    let email=req.body.email;
    let password=req.body.password;
    let sql="SELECT * FROM user WHERE Username = ?";
    db.query(sql,[username],(err,rows,fields)=>{
        if(rows.length>0){
            res.send("Username Already Taken");
        }else {
            sql="INSERT INTO `elibrary`.`user` (`Username`, `Name`, `Contact`, `Email`, `Password`) VALUES (?, ?, ?, ?, ?); INSERT INTO `elibrary`.`members` (`Username`, `Download`, `Spent`) VALUES (?, '0', '0');";
            db.query(sql,[username,fullname,contact,email,password,username]);
            res.redirect('/');
        }
    });
    
    
});

app.post('/search',(req, res) => {
     req.session.keyword=req.body.keyword;
     req.session.key=true;
     
     res.redirect('/home');

});
app.post('/nosearch',(req, res) => {
    req.session.keyword=req.body.keyword;
    req.session.key=true;
    
    res.redirect('/user');

});
app.get('/popular',(req,res)=>{
    let sql="Select book.Book_name, count(added.Book_id) as num from added join book on book.id=added.Book_id group By book.id order by num desc;";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('popular',{
            rows:rows
        });
    });
});

app.get('/author',(req,res)=>{
    let sql="Select * from author";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('author',{
            rows:rows
        });
    });
});
app.get('/noauthor',(req,res)=>{
    let sql="Select * from author";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('noauthor',{
            rows:rows
        });
    });
});
app.get('/noabout',(req,res)=>{
    let sql="Select Name from user join admin where user.Username=admin.Username";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('noabout',{
            rows:rows
        });
    });
});
app.get('/sales',(req,res)=>{
    let sql="SELECT Id as Cart_id, DATE_FORMAT(Date,'%y/%m/%d') as date, Total as Revenue FROM elibrary.payment;";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('sales',{
            rows:rows
        });
    });
});
app.get('/demand',(req,res)=>{
    res.render('demand');
});
app.get('/book',(req,res)=>{
    
    
    res.render('book');
    
});
app.get('/list',(req,res)=>{
    let sql="SELECT * from demand;";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        res.render('list',{
            rows:rows
        });
    });
});
app.post('/need',(req,res)=>{
    
    let id,version,authorname,bookname;
    
    version=req.body.version;
   
    authorname=req.body.authorname;
    bookname=req.body.bookname;
    

    let sql="SELECT * from demand";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        id=rows.length+1;
        

        db.query(sql,(err,rows,fields)=>{
                if(err) throw err;
                
                sql="INSERT INTO `elibrary`.`demand` (`Serial`, `Author`, `Version`, `Book_name`) VALUES (?, ?, ?, ?);";
                db.query(sql,[id,authorname,version,bookname]);
                    
                    
                res.redirect('/user');
              

            
        });

            

    });
    
    
});
app.post('/add',(req,res)=>{
    
    let id,price,version,size,authorname,bookname,auth_id,pubname;
    price=req.body.price;
    version=req.body.version;
    size=req.body.size;
    authorname=req.body.authorname;
    bookname=req.body.bookname;
    pubname=req.body.pubname;

    let sql="SELECT * from book";
    db.query(sql,(err,rows,fields)=>{
        if(err) throw err;
        id=rows.length+1;
        sql="SELECT * from author";

        db.query(sql,(err,rows,fields)=>{
                if(err) throw err;
                let i=rows.length;
                sql="SELECT * from author where name=?";
                my="INSERT INTO `elibrary`.`book` (`id`, `Price`, `Version`, `Size`, `Author`, `Book_name`, `Author_id`, `Publisher_name`, `Username`) VALUES (?,?,?,?,?,?,?,?,?); ";
                db.query(sql,[authorname],(err,rows,fields)=>{
                if(err) throw err;
                    if(rows.length>0){
                        auth_id=rows[0].Id;
                        console.log(rows);
                    }else {
                        auth_id=(i+1);
                        
                    }
                    
                    my+="INSERT INTO `elibrary`.`author` (`Name`, `Id`, `Genre`) SELECT * FROM (SELECT ?, ?, 'any') as temp where not exists ( select * from author where name=?) limit 1;";
                    my+="INSERT INTO `elibrary`.`written_by` (`author_Id`, `book_id`) VALUES (?, ?)";
                    
                    db.query(my,[id,price,version,size,authorname,bookname,auth_id,pubname,req.session.name,authorname,auth_id,authorname,auth_id,id]);
                    
                    res.redirect('/home');
                });

          
        });

            

    });
    
    
});

app.listen('3300', ()=>{
    console.log("server started");
});