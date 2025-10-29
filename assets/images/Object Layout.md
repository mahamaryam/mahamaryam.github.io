'''
class Base
{
public:
    int a;
    char c;

    Base(int a, char c)
    {
        this->a = a;
        this->c = c;
    }
};

int main()
{
    Base b1(5, 'a');
    Base *b2 = new Base(9, 'x');

    return 0;
}
'''